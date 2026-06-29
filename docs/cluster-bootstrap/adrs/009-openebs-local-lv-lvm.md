# ADR-009: OpenEBS LocalLV LVM for low-latency local storage

**Status:** Accepted — **selectable, Talos-coupled feature** (`features.openebs`; see ADR-012). Enabling it renders both the Flux HelmRelease *and* the Talos disk/volume config (ADR-013), so it requires an Ansible render + bootstrap, not a pure-git toggle.

## Context

Network-attached block storage (e.g. Hetzner CSI) adds latency from the network hop and the storage backend. Latency-sensitive workloads (databases, message queues) benefit from storage that lives on the node's own disk. A second storage tier on local disks is needed alongside the networked tier.

## Decision

Deploy **OpenEBS LocalLV LVM** as the local storage tier. OpenEBS manages an LVM Volume Group on a dedicated disk partition on each node and provisions PVCs as LVM Logical Volumes directly on that VG. This gives local-disk performance with Kubernetes-native PVC semantics.

The block device for the LVM VG is provisioned at **Talos bootstrap time** via a `RawVolumeConfig` (raw block device — LVM needs an unformatted device, not a mounted `UserVolume`; see ADR-013 and talos#11847) selected by `diskSelector`. This is a required step in `bootstrap-talos.yml` / the `talos_config` role, not a post-install manual step. (`machine.disks` is deprecated in Talos v1.13.)

## Consequences

- **Local storage is not replicated** — if the node is lost, the data on its local PVCs is lost. Workloads using this tier must handle their own replication (e.g. application-level replication, regular backups) or accept the risk.
- **Pod is pinned to the node** — LocalLV PVCs are node-affinity-bound; a pod using a local PVC cannot be rescheduled to a different node. Plan for this in workload design.
- **Talos disk layout must be planned before first boot.** The volume's `diskSelector`/`minSize` is set in `cluster.yaml` `storage.volumes`; changing the layout later requires wiping and re-provisioning the node. Size the LVM device with headroom.
- **Two storage classes in the cluster:** `hcloud-volumes` (network-attached, portable, replicated by Hetzner) and `openebs-lvm-localpv` (local, fast, node-pinned). Workload authors choose based on latency vs durability requirements.
- Chose OpenEBS LocalLV LVM over `hostPath` (no PVC lifecycle management), `local-path-provisioner` (no LVM thin-provisioning or snapshot support), and Rook/Ceph (heavy, full distributed storage stack — overkill for a 3-node homelab).

## Addendum (June 2026): Implementation decisions

### Context

ADR-009 specified the what (RawVolumeConfig, storage.volumes schema, StorageClass name, two-tier storage) but left the how unspecified: VG provisioning mechanism, device targeting, and kernel-module loading. The implementation made these decisions.

### Decision

#### 1. VG provisioning via privileged initContainer on the node DaemonSet

The lvm-localpv Helm chart does NOT expose an initContainers values hook. Rather than forking the chart or running a separate DaemonSet, the VG is created by a **privileged initContainer** injected through a Flux HelmRelease `spec.postRenderers` [kustomize strategic-merge patch](https://fluxcd.io/flux/cheatsheets/helm-post-renderer/).

The init container:

- Runs on every node via the DaemonSet (initContainer before the main CSI node-plugin starts)
- Is idempotent: checks `vgs <vgName>` first; if absent, checks `pvs` for partial-state recovery, then `pvcreate` + `vgcreate`
- Fails loudly if the Talos partition device is missing — never targets a blank/wrong device
- Uses the same `openebs/lvm-driver` image (shipped with lvm2) as the node plugin, with LVM locking disabled (`locking_type=0`) for the one-shot init container only
- Uses the `openebs-vg` volume group name, hardcoded in the Flux catalog (StorageClass `volgroup` + init-container `vgcreate`)

#### 2. Boot disk partition via `RawVolumeConfig`

With kube1 nodes being 3× CX33 Hetzner instances (single ~80GB NVMe boot disk), the raw volume is carved from **free space on the system/boot disk** using a Talos `RawVolumeConfig` with:

```yaml
diskSelector:
  match: system_disk    # CEL boolean — targets the boot disk
minSize: 20GB
maxSize: 20GB            # fixed-size, no grow
```

The partition surfaces at `/dev/disk/by-partlabel/r-<name>` (here `r-openebs-lvm`). `minSize == maxSize` ensures a deterministic fixed partition; `grow: false` by omission.

A dedicated data disk would be a future improvement for larger capacity, but the CX33 form factor has no second disk slot.

#### 3. Device-mapper kernel modules

OpenEBS LVM requires `dm_snapshot` and `dm_thin_pool` (thin provisioning is enabled) loaded on the Talos host. These are loaded via an **all-node Talos `machine.kernel.modules` patch** (gated on `features.openebs: true`):

```yaml
machine:
  kernel:
    modules:
      - name: dm_mod
      - name: dm_snapshot
      - name: dm_thin_pool
```

#### 4. CSI namespace convention

Storage CSI drivers introduced a new namespace convention: `csi-<name>-system` (e.g. `csi-openebs-system`), grouping all storage CSI by purpose. This extends the existing dedicated-`<component>-system` pattern used by `cilium-system` and `hcloud-ccm-system` (which are network/cloud-provider components, not CSI drivers).

### Consequences

**Positive:**

- Clean Talos↔Flux separation: Ansible owns the partition + module loading; Flux owns the CSI driver + VG creation.
- Self-contained catalog entry with no Ansible pre-install (unlike cilium/ccm).
- The init-container approach keeps VG provisioning co-located with the CSI driver and handles all edge states (re-run, partial failure, wrong device).

**Negative:**

- Boot-disk partition (20GB) is small and not resizable in place — changing capacity requires wiping and re-provisioning.
- The known Talos+storage edge cases (talos#13354 shutdown hang with active LVs, talos#9300 boot-loop with thin LVs) apply and are documented in the runbook.

**Maintenance:**

- The init-container image tag is a literal in the postRenderer patch, decoupled from the Helm value `lvmPlugin.image.tag` — must be bumped in lockstep (annotated with `🛠 Keep in sync` comments in the source).
