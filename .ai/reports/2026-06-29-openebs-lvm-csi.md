# Report: OpenEBS LVM LocalPV implementation

**Date:** 2026-06-29
**Plan:** `.ai/plans/2026-06-29-openebs-lvm-csi.md`
**Status:** Built, reviewed, validated. **Not yet observed on a live cluster** (per CLAUDE.md "live-unverified" caveat — `bootstrap-flux.yml` has not been run against kube1 yet, so we cannot confirm the partition / VG / DaemonSet init-container actually works on Talos+kernel).

---

## What changed

### Config inputs (`config/`)
- `config/schemas/cluster-input.schema.json` — `storage.volumes` replaced with a structured item schema (`name`, `type` enum `[raw]`, `diskSelector`, `minSize`, `maxSize`, `openebs` boolean) + `additionalProperties: false` on each item. Structurally rejects any LUKS/`encryption` key (talos#13354).
- `config/clusters/kube1/cluster.yaml` — `features.openebs: true`; added one `storage.volumes` entry: `name: openebs-lvm`, `type: raw`, `diskSelector: system_disk`, fixed 20 GB (`minSize == maxSize`), `openebs: true`. Inline comments record the boot-disk-partition rationale and Talos↔Flux coupling.
- `config/defaults/cluster.yaml` — unchanged values (`openebs: false`, empty volumes); added a clarifying comment about the schema-enforced shape.

### Render compiler (`infra/ansible/roles/config_render/`)
- `tasks/validate.yml` — replaced the deferred `storage.volumes` guard with real validation: name pattern + uniqueness, `type == raw` required when `features.openebs`, reject `encryption`/`luks` keys, enforce `openebs ⇒ raw` rule. Pre-existing `talos.install.disk` / `install.diskSelector` guards left untouched.
- `tasks/compute.yml` — added `config_render_flux_features` (post-bootstrap/Flux-managed catalog list — `[openebs]` when enabled) and `config_render_storage_raw_volumes` (filter of volumes with `type == raw`). Kept separate from the pre-Flux `bootstrap_pre_flux_components` list.
- `tasks/render_talos.yml` — paired copy/remove tasks for the two new generated all-node files, gated on `config_render_openebs_enabled`. Toggle off → both files removed cleanly.
- `templates/talos_patches/all/25-kernel-modules.yaml.j2` *(new)* — `machine.kernel.modules: [dm_mod, dm_snapshot, dm_thin_pool]`, strategic-merge into the v1alpha1 doc.
- `templates/talos_patches/all/30-raw-volumes.yaml.j2` *(new)* — one `RawVolumeConfig` document per raw volume, separated with `---`. Emits `name`, `provisioning.diskSelector.match`, `minSize`, `maxSize`.
- `tasks/render_flux.yml` — appends `../../_components/openebs` to the generated `selected/kustomization.yaml` when `openebs in config_render_flux_features`.

### Talos apply (`infra/ansible/roles/talos_config/`)
- `tasks/main.yml` — patch-ordering comment block updated to list the two new all-node files. The role's existing glob over `generated/all/*.yaml.j2` picks them up automatically — no apply-path change.

### Flux catalog base (`flux/infrastructure/controllers/_components/openebs/`, *new*)
Ordinary kustomize base (5 files), per `_components/README.md` contract:
- `namespace.yaml` — `Namespace csi-openebs-system` (new repo convention: CSI drivers get a `csi-<name>-system` namespace; future Hetzner CSI → `csi-hcloud-system`).
- `helmrepository.yaml` — `https://openebs.github.io/lvm-localpv`, `interval: 1h`.
- `helmrelease.yaml` — chart `lvm-localpv` pinned to **1.9.1** (released 2026-06-10); `lvmPlugin.image.tag` pinned to **1.9.1** (no `:latest` / no `-develop`); `targetNamespace: csi-openebs-system` + `install.createNamespace: true`; full toleration list on `lvmNode` including the `node.cloudprovider.kubernetes.io/uninitialized` key (same external-cloud-provider concern as Cilium #41921); `spec.postRenderers` kustomize strategic-merge patch injecting the privileged **VG-provisioning init container** onto the `openebs-lvm-localpv-node` DaemonSet. The init container is idempotent: `vgs openebs-vg` succeeds → exit 0; else verify `/dev/disk/by-partlabel/r-openebs-lvm` exists and **fail loudly** if missing, then `pvcreate` + `vgcreate`.
- `openebs-lvm-localpv.yaml` — `StorageClass`: `provisioner: local.csi.openebs.io`, `parameters: { storage: "lvm", volgroup: "openebs-vg", thinProvision: "yes" }`, `volumeBindingMode: WaitForFirstConsumer`, `allowVolumeExpansion: true`. Thin-provisioning enabled (decision §1a #3).
- `kustomization.yaml` — `resources: [namespace, helmrepository, helmrelease, openebs-lvm-localpv]`.
- `.yamllint.yaml` — local relax for the multi-doc postRenderer patch.

Purely Flux-managed — no pre-Flux Ansible install, no adoption handshake (unlike cilium/hcloud-ccm).

### Generated outputs (`flux/`, `infra/talos/`)
- `flux/infrastructure/controllers/generated/selected/kustomization.yaml` — now lists `../../_components/openebs` alongside cilium and hcloud-ccm.
- `infra/talos/patches/generated/all/25-kernel-modules.yaml.j2` *(new)* and `infra/talos/patches/generated/all/30-raw-volumes.yaml.j2` *(new)* — first-render output of the new templates.

### Docs (`docs/`)
- `docs/cluster-bootstrap/adrs/009-openebs-local-lv-lvm.md` — appended addendum recording the four implementation decisions ADR-009 left unspecified: (1) VG provisioning via privileged init container + HelmRelease `postRenderers` (chart has no `initContainers` hook), (2) boot-disk partition via `RawVolumeConfig diskSelector: system_disk`, (3) device-mapper kernel-module requirement, (4) `csi-<name>-system` namespace convention.
- `docs/openebs-local-storage.md` *(new)* — full day-2 runbook: enable, verify (`talosctl get volumestatus`, `ls -l /dev/disk/by-partlabel/r-openebs-lvm`, `kubectl -n csi-openebs-system get pods`), PVC test, resize (requires wipe/re-provision per ADR-009), reboot/upgrade caveats (talos#13354 LUKS+LVM + talos#9300 thin-LV boot-loop), troubleshooting.
- `flux/infrastructure/controllers/_components/README.md` — current-state list updated with the openebs entry + the `csi-<name>-system` namespace convention.
- `CLAUDE.md` — Implementation status updated: OpenEBS moved from "Still deferred" to "Working — built/reviewed/validated, **live-unverified**" with a short summary and a pointer to the runbook.

---

## What was validated

| Gate | Result |
|---|---|
| `ansible-lint` (all changed task files) | 0 failures, 0 warnings |
| `playbooks/render-config.yml` — first run | 3 new files rendered (2 Talos patches + `selected/kustomization.yaml` updated) |
| `playbooks/render-config.yml` — second run | 0 changed (idempotent) |
| `-e config_render_check=true` (check mode) | "no drift" — toggle-off path also clean (paired remove) |
| `yamllint flux/infrastructure/controllers/_components/openebs/` | 0 errors |
| `kustomize build flux/infrastructure/controllers/_components/openebs` | clean — 4 resources (Namespace, HelmRepository, HelmRelease, StorageClass) |
| `helm template` with pinned chart + release values | confirms tolerations/pins/DaemonSet name (`openebs-lvm-localpv-node`) |
| `kubeconform` on kustomize output | clean (skipping Helm CRDs) |
| `ansible-reviewer` | **APPROVED** (1 LOW comment applied) |
| `k8s-reviewer` | **APPROVED** after 1 MEDIUM fix round — partial-failure idempotency in the init-container shell script (the `vgs` guard now exits cleanly on success, and the create sequence is guarded so a half-applied `pvcreate`/`vgcreate` does not leave the VG in an inconsistent state on retry) |

---

## What was NOT run and why

- **No `ansible-playbook -i inventories/providers/hcloud playbooks/bootstrap-talos.yml`.** Live cluster apply. The user runs Ansible against the real cluster in their own terminal; agents do not.
- **No `flux reconcile`.** Same reason.
- **No `talosctl get volumestatus` / `kubectl -n csi-openebs-system get pods`.** No live cluster.
- **Result:** "done" here means "the code is built, lints, renders idempotently, builds cleanly under `kustomize` + `helm template` + `kubeconform`, and passes both reviewers". The runbook's verify steps have not been observed executing against a real Talos + kernel.

---

## Residual risks (unchanged from plan §8, all tracked)

- **R1 — `RawVolumeConfig` via `gen config --config-patch`.** Strongly expected to work (the 90-user-raw `HostnameConfig` precedent already rides a separate document in a `--config-patch`). Falls back to a `talosctl patch mc` step in `talos_config` if the live apply rejects it — larger change.
- **R2 — node-DaemonSet name.** Confirmed as `openebs-lvm-localpv-node` via `helm template` against the pinned chart + releaseName.
- **R3 — Talos↔Flux coupling.** The partlabel `r-openebs-lvm` (from `config/clusters/kube1/cluster.yaml` volume `name`) is the sole coupling point between the Talos-generated RawVolumeConfig and the Flux-side init container. Documented in runbook §9 and ADR-009 addendum. The vgName `openebs-vg` is hardcoded in the catalog and is the second coupling point — `volgroup` and the init container's `vgcreate` are hand-kept in lock-step; reviewer-checked; no schema wiring.
- **Q4 — chart version.** Pinned to `lvm-localpv` **1.9.1** at implementation time (released 2026-06-10); `lvmPlugin.image.tag` pinned to the matching `1.9.1`. Both pins annotated in the HelmRelease.

---

## Open items / follow-ups

1. **Live-apply sequence — user runs:**
   - `cd infra/ansible && ansible-playbook playbooks/render-config.yml` — review git diff.
   - `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml` — applies the RawVolumeConfig (carves the `r-openebs-lvm` partition) and the kernel-module patch. May require a reboot; if so, run `upgrade-talos.yml`. Verify with `talosctl get volumestatus r-openebs-lvm` and `talosctl read /proc/modules | grep dm_`.
   - `flux reconcile kustomization flux-system --with-source` — Flux deploys the openebs catalog; the init container creates the PV+VG on each node. Verify per the runbook.
2. **`csi-<name>-system` namespace convention is now documented** in the ADR-009 addendum and `_components/README.md`. Future CSI drivers (Hetzner CSI first) should follow it — and their catalog entries should also be `csi-hcloud-system`/`csi-<name>-system` to keep the storage-CSI grouping consistent. Future enhancement: a schema-enforced `csiName` → namespace convention, deferred.
3. **Thin provisioning is enabled** on the StorageClass and `dm_thin_pool` is loaded. The talos#9300 boot-loop caveat is documented in the runbook's "Reboot/upgrade caveats" section; recovery is a wipe/re-provision of the LV.
4. **Init-container image coupling** — the init container reuses the `openebs/lvm-driver` image (already pulled by the chart, ships LVM2 userspace). The image tag in the postRenderer is **pinned independently of the Helm value** (both annotated with `🛠 Keep in sync` comments). A maintainer bumping the chart's `lvmPlugin.image.tag` must also bump the postRenderer image tag in the same commit.
5. **Talos version pin** — still v1.13.3; no change.
6. **Live-unverified caveat remains.** The next agent (or the user, on first apply) should confirm: `talosctl get volumestatus r-openebs-lvm` shows the partition, `/dev/disk/by-partlabel/r-openebs-lvm` exists, and `kubectl -n csi-openebs-system get pods` shows the node DaemonSet Ready with the init container Completed. If any of those fail, file a follow-up — the schema/config choices are clean, so failures will be Talos/kernel/boot-disk-shape issues, not schema bugs.
