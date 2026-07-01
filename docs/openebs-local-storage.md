# OpenEBS LVM LocalPV

Operational runbook for the local-storage tier on this template. The architectural
background and trade-offs (local vs networked storage, the `RawVolumeConfig`
+ LVM design, the `hcloud-volumes` vs `openebs-lvm-localpv` choice) live in
[ADR-009](../cluster-bootstrap/adrs/009-openebs-local-lv-lvm.md) — this
document is the day-2 procedure.

## Overview

OpenEBS LVM LocalPV is a local-storage CSI driver that backs PVCs with LVM
Logical Volumes on a host-local Volume Group — local-disk latency with
Kubernetes-native PVC semantics. This repo provisions the backing device via
a Talos `RawVolumeConfig` partition on the system/boot disk and uses a
privileged init container on the lvm-node DaemonSet to create the LVM VG
(`openebs-vg`) on that partition. The example cluster is a 3-hybrid-node cluster, all on a
single boot NVMe — the same 20GB partition shape backs all three node-local VGs.

## How it works

1. **Talos `VolumeConfig` + `RawVolumeConfig`** first caps EPHEMERAL so the
   system disk keeps free space, then carves a 20GB raw partition from the
   remaining space (the boot NVMe). The raw volume surfaces at the stable symlink
   `/dev/disk/by-partlabel/r-openebs-lvm` — the partlabel is auto-derived as
   `r-<name>` from the volume `name: openebs-lvm`. Source:
   `infra/talos/patches/generated/all/30-raw-volumes.yaml.j2` (render-generated,
   gated on `features.openebs`).
2. **Device-mapper kernel modules** (`dm_mod`, `dm_snapshot`, `dm_thin_pool`)
   are declared in `machine.kernel.modules` so the lvm-localpv CSI driver and
   thin provisioning work. Source:
   `infra/talos/patches/generated/all/25-kernel-modules.yaml.j2` (render-generated).
3. **A privileged `vg-provision` init container** on the `openebs-lvm-localpv-node`
   DaemonSet runs idempotently:
   - `vgs openebs-vg` exists → exit 0 (no-op, common on a re-roll).
   - Partlabel missing → `FATAL` exit 1 (Talos partition absent — the operator
     must re-run `bootstrap-talos.yml`; the init container never touches a
     wrong/blank device).
   - PV exists, VG missing → `vgscan` and recover.
   - PV belongs to a different VG → `FATAL` exit 1 (manual intervention).
   - Fresh device → `pvcreate` + `vgcreate openebs-vg`.
   The init container is injected via a Flux HelmRelease `spec.postRenderers`
   kustomize strategic-merge patch — the upstream `lvm-localpv` chart exposes
   no native `initContainers` hook.
   If the partlabel is missing, the usual cause is that Talos never got the
   capped EPHEMERAL layout on first boot; rebuild the node, do not just reboot.
4. **The OpenEBS lvm-localpv CSI provisioner** creates LVM logical volumes from
   `openebs-vg` for PVCs that use StorageClass `openebs-lvm-localpv`
   (thin-provisioned, `WaitForFirstConsumer`, `allowVolumeExpansion: true`).
5. **The driver runs in `csi-openebs-system`** (new CSI-driver namespace
   convention from `_components/README.md`).

> **Two storage classes in the cluster:** `hcloud-volumes` (network-attached,
> portable, replicated by Hetzner) and `openebs-lvm-localpv` (local, fast,
> node-pinned). Workload authors choose based on latency vs durability. See
> ADR-009 for the trade-off.

## Enabling

### 1. Toggle the cluster config

OpenEBS is **already enabled** in `config/clusters/<cluster>/cluster.yaml`. The
two enablement points are:

```yaml
features:
  openebs: true

storage:
  volumes:
    - name: openebs-lvm
      type: raw
      diskSelector: system_disk
      minSize: 20GB
      maxSize: 20GB
      openebs: true
```

To **disable**, set `features.openebs: false` and re-render. The render's
paired copy/remove tasks will remove the generated Talos patches and the
Flux catalog reference, so toggling cleanly wipes the artifacts.

> **Talos↔Flux coupling (single point):** the volume `name: openebs-lvm` flows
> to the partlabel `r-openebs-lvm` (Talos) which the init container hardcodes
> (Flux). Renaming the volume requires re-rendering AND updating the
> `DEV=/dev/disk/by-partlabel/r-<new-name>` line in the
> `helmrelease.yaml` postRenderer shell script. See §9.

### 2. Re-render the generated tree

```bash
cd infra/ansible
ansible-playbook playbooks/render-config.yml
```

Review the git diff. New generated artifacts:

| File | Contents |
|---|---|
| `infra/talos/patches/generated/all/25-kernel-modules.yaml.j2` | `machine.kernel.modules` listing `dm_mod`, `dm_snapshot`, `dm_thin_pool` |
| `infra/talos/patches/generated/all/30-raw-volumes.yaml.j2` | `VolumeConfig` for `EPHEMERAL` cap + `RawVolumeConfig` with `name: openebs-lvm`, `diskSelector.match: system_disk`, `minSize/maxSize: 20GB` |
| `flux/infrastructure/controllers/generated/selected/kustomization.yaml` | gains `../../_components/openebs` alongside `cilium` and `hcloud-ccm` |

Use `-e config_render_check=true` to render to a temp dir and diff without
writing — clean check-mode diffs confirm toggle symmetry.

### 3. Apply the Talos patches

```bash
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml
```

This applies the EPHEMERAL cap + RawVolumeConfig (carves `r-openebs-lvm` from
the system disk) and the kernel-module patch.

> **A reboot is not enough** if the raw volume failed because EPHEMERAL filled
> the disk. The partition layout only changes on a fresh Talos install, so
> rebuild the node and re-run `bootstrap-talos.yml`. Reboot only helps once the
> partition already exists and Talos is finishing the module load.

Verify the partition and modules:

```bash
talosctl --talosconfig infra/talos/secrets/talosconfig get volumestatus r-openebs-lvm
talosctl -n <node> read /proc/modules | grep -E 'dm_mod|dm_snapshot|dm_thin_pool'
```

Expected: `STATUS=Ready`, `SIZE=20GB`; all three `dm_*` modules present on
every node.

### 4. Reconcile Flux

```bash
flux reconcile kustomization flux-system --with-source
```

Or wait for the default 1h interval. Verify:

```bash
kubectl -n csi-openebs-system rollout status daemonset/openebs-lvm-localpv-node
kubectl get sc openebs-lvm-localpv
kubectl -n csi-openebs-system logs daemonset/openebs-lvm-localpv-node --container vg-provision
```

Expected: DaemonSet `Ready`, the init container logged
`VG openebs-vg provisioned.` (or exited 0 on a re-run because the VG
already exists).

## Verification

After step 4, run the full health check:

```bash
# All 3 nodes host an lvm-node pod.
kubectl get pods -n csi-openebs-system -o wide

# The Talos partition is provisioned on every node.
talosctl -n <node> get volumestatus r-openebs-lvm

# The LVM volume group exists on a node (from inside the plugin container,
# which runs privileged and has the host /dev mounted).
kubectl -n csi-openebs-system exec -it daemonset/openebs-lvm-localpv-node \
  -c openebs-lvm-plugin -- vgs openebs-vg

# The StorageClass is in place with thin provisioning + WaitForFirstConsumer.
kubectl describe sc openebs-lvm-localpv
```

Expected output (per node):

- `openebs-lvm-localpv-node-xxxx` pod `Running` on each `example-hbN`.
- `volumestatus r-openebs-lvm` → `STATUS=Ready`, `SIZE=20GB`.
- `vgs openebs-vg` shows one `PV` (`/dev/disk/by-partlabel/r-openebs-lvm`).
- StorageClass: `Provisioner: local.csi.openebs.io`, `Parameters:
  storage=lvm, volgroup=openebs-vg, thinProvision=yes`, `VolumeBindingMode:
  WaitForFirstConsumer`, `AllowVolumeExpansion: true`.

If any check fails, do **not** proceed to install workloads.

## Provisioning a test PVC

Minimal copy-paste example. The `nodeName` field pins the pod to one node —
this is **required** for `WaitForFirstConsumer` to resolve, since the StorageClass
needs a node identity before it can pick a VG:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openebs-lvm-test
spec:
  storageClassName: openebs-lvm-localpv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: openebs-lvm-test
spec:
  nodeName: example-hb1                # pin to one of the three hybrid nodes
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "echo hello > /data/hello && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: openebs-lvm-test
```

```bash
kubectl apply -f openebs-lvm-test.yaml
kubectl get pvc openebs-lvm-test       # STATUS should reach Bound
kubectl exec openebs-lvm-test -- cat /data/hello
```

`WaitForFirstConsumer` is deliberate: the pod's node selector (or `nodeName`
like above) determines where the LV is provisioned, which is the only way
local-PV semantics work. PVCs with no scheduled pod stay `Pending` — that is
correct, not a bug.

## Resizing

### PVC resize (LVM logical volume)

The StorageClass has `allowVolumeExpansion: true`, so a thin LV can grow in
place:

```bash
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
```

This is a **logical volume** expansion — it does not touch the Talos
`RawVolumeConfig` partition, which stays at 20GB. The LV grows within the
VG's free extents; if the underlying volume group itself runs out, the
expansion fails (recover by deleting unused LVs or, in extremis, see the
partition resize procedure below). The Talos EPHEMERAL cap is what leaves the
raw partition free space in the first place.

### Partition resize (RawVolumeConfig) — non-trivial

**Not supported in place.** A `RawVolumeConfig` is provisioned once and the
partition is fixed. Resizing the partition is a destructive operation:

1. **Drain the node** — `kubectl drain <node> --ignore-daemonsets
   --delete-emptydir-data`. This is required even for `ReadWriteOnce` PVCs
   because the lvm-node DaemonSet pods keep the VG open.
2. **Deactivate the VG** from a privileged pod:
   ```bash
   kubectl -n csi-openebs-system exec -it daemonset/openebs-lvm-localpv-node \
     -c openebs-lvm-plugin -- vgchange -an openebs-vg
   ```
3. **Back up or delete** any PVCs using LVs from this node. The LVs are
   destroyed with the VG.
4. **Wipe the partition:**
   ```bash
   kubectl -n csi-openebs-system exec -it daemonset/openebs-lvm-localpv-node \
     -c openebs-lvm-plugin -- sh -c \
     'vgremove -y openebs-vg && pvremove /dev/disk/by-partlabel/r-openebs-lvm'
   talosctl -n <node> wipe /dev/disk/by-partlabel/r-openebs-lvm
   ```
5. **Update the Talos storage sizing** in `config/clusters/<cluster>/cluster.yaml`
   (or the generated Talos storage template) and re-render.
6. **Rebuild the node, then re-run `bootstrap-talos.yml`** — EPHEMERAL and
   RawVolumeConfig sizing are only honored on initial disk provisioning.
7. **Re-run the Flux reconciliation** — the init container recreates the VG
   on the resized partition.
8. **Re-create the PVCs and reschedule the workloads.**

Plan for this; do not casually change the partition size.

## Reboot / upgrade caveats

These are known Talos + local-storage stacking edge cases, not unique to
this setup. Both are documented upstream.

### Active LVs can block shutdown (talos#13354)

Even without LUKS, active LVM volumes can keep device-mapper references that
prevent Talos from unmounting cleanly on shutdown. **Before any node reboot
or Talos upgrade:**

1. Evict all pods using `openebs-lvm-localpv` PVCs on that node
   (`kubectl drain` or targeted `kubectl delete pod`).
2. Deactivate the volume group:
   ```bash
   kubectl -n csi-openebs-system exec -it daemonset/openebs-lvm-localpv-node \
     -c openebs-lvm-plugin -- vgchange -an openebs-vg
   ```
3. Then run the upgrade / reboot.

If you forget, the node may hang on shutdown and require a hard power-cycle.

### Kernel-module load race with thin LVs (talos#9300)

A documented Talos caveat: thin/raid LVs combined with the `dm_*` module
load race can cause Talos to fail to boot, even without LUKS. We are not
using raid LVs in this repo, but the thin pool is enabled. **If a node
fails to boot after enabling OpenEBS:**

1. Reboot into Talos maintenance mode.
2. Wipe the affected partition and VG:
   ```bash
   vgchange -an openebs-vg
   vgremove -y openebs-vg
   pvremove /dev/disk/by-partlabel/r-openebs-lvm
   wipefs -a /dev/disk/by-partlabel/r-openebs-lvm
   ```
3. Reboot normally; the init container recreates the VG.
4. See the Talos recovery docs (docs.siderolabs.com) for the exact
   maintenance-mode procedure.

## Troubleshooting

- **`FATAL: /dev/disk/by-partlabel/r-openebs-lvm not found`** in the
  `vg-provision` init container: the Talos `RawVolumeConfig` hasn't been
  applied, or the partition didn't provision. Re-run `bootstrap-talos.yml`;
  verify with `talosctl get volumestatus r-openebs-lvm`.
- **`FATAL: /dev/... belongs to VG 'other-vg' not 'openebs-vg'`**: the
  partition was previously used by a different LVM setup. Manual
  intervention — drain the node, deactivate the other VG, then `pvremove`
  the partition before re-bootstrapping.
- **PVC stuck in `Pending`**: check that `openebs-vg` exists on the target
  node (`vgs openebs-vg` via the plugin container); confirm the StorageClass
  `parameters.volgroup` matches the VG name (`openebs-vg`); remember
  `volumeBindingMode: WaitForFirstConsumer` — the workload pod must be
  scheduled first, and the pod's `nodeName` / node selector determines where
  the LV is created.
- **PV create succeeds but pod fails to mount**: check `talosctl dmesg` on
  the affected node for device-mapper errors; confirm `dm_snapshot` and
  `dm_thin_pool` are loaded (`talosctl read /proc/modules | grep dm_`).
- **Talos reports a reboot is needed after `bootstrap-talos.yml`**: expected
  — run `upgrade-talos.yml` to roll it, or `talosctl reboot`.
- **`openebs-lvm-localpv-node` pods CrashLoopBackOff**: check the
  `vg-provision` init container logs first. If the init container exited 0
  but the main container is failing, the issue is downstream (image pull,
  RBAC, node-driver-registrar).

## OpenEBS-related GitOps config

- **Catalog lives in** `flux/infrastructure/controllers/_components/openebs/`.
  Each file is template-maintained; cluster-specific overrides go under
  `../../overlays/clusters/<cluster>/` (the standard per-cluster overlay path).
- **Talos↔Flux coupling (single point):** the StorageClass
  `parameters.volgroup` and the init-container `vgName` are both hardcoded
  to `openebs-vg`. To change the VG name, edit BOTH:
  - `helmrelease.yaml` — the `VG=openebs-vg` line in the postRenderer shell script
  - `openebs-lvm-localpv.yaml` — `parameters.volgroup`
  And if the partlabel changes, also update the `name: openebs-lvm` volume
  in `config/clusters/<cluster>/cluster.yaml` and re-render.
- **Image pins** — both `lvmPlugin.image.tag` and the postRenderer
  `docker.io/openebs/lvm-driver:<tag>` are pinned in lockstep to the chart
  version (currently `1.9.1`). The chart's `appVersion` does not always
  match the chart release, so both pins are explicit. On a chart version
  bump, update BOTH.
- **Chart version** — pinned to `lvm-localpv` `1.9.1`. See the
  [OpenEBS lvm-localpv releases](https://github.com/openebs/lvm-localpv/releases)
  for the latest stable.
- **Snapshot support** — the chart's `crds.csi.volumeSnapshots.enabled:
  true` is set because thin provisioning is enabled. The chart embeds its
  own snapshot-controller; no external `volumesnapshot.snapshot.storage.k8s.io`
  CRDs are required. If a cluster-wide snapshot stack is added later, this
  must be disabled to avoid ownership conflicts.
- **Architectural background** — see
  [ADR-009](../cluster-bootstrap/adrs/009-openebs-local-lv-lvm.md) (the
  LocalLV LVM decision) and ADR-012 / ADR-013 (feature catalog, layered
  Talos config).
