# Plan: OpenEBS LocalLV LVM (Talos-coupled feature) for kube1

**Date:** 2026-06-29
**Slug:** openebs-lvm-csi
**Status:** Planned — not yet implemented
**Related ADRs:** ADR-009 (OpenEBS LocalLV LVM), ADR-012 (feature catalog / Talos-coupled taxonomy), ADR-013 (layered Talos config — `storage.volumes`, `openebs ⇒ raw`), ADR-016 (Talos config boundary / selected-cluster source), ADR-017 (shared overlay ownership)
**Live state:** UNVERIFIED. This plan produces built + reviewed + validated code. "Done" means it renders cleanly, lints, builds, and passes review — NOT that it has been observed running on a live cluster (CLAUDE.md unverified-live-cluster caveat).

---

## 1. Goal & scope

Land the currently-DEFERRED `features.openebs` feature for kube1 end-to-end:

- Turn the hard-fail `storage.volumes` deferred guard into **real structured validation**.
- Emit a Talos **`RawVolumeConfig`** (partition carved from free space on the **system/boot disk**) + a **device-mapper kernel-module** patch into the generated all-node Talos patch tree, gated on `features.openebs`.
- Emit the OpenEBS **Flux catalog entry** into `generated/selected/` via a NEW Flux-managed (post-bootstrap) selected-features list.
- Build the self-contained `_components/openebs/` catalog base: Namespace + HelmRepository + HelmRelease + StorageClass, with the **VG-provisioning init container injected via a HelmRelease `spec.postRenderers` strategic-merge patch** (the chart has no native initContainers hook — confirmed below).
- Document the runbook; update CLAUDE.md status; add a short ADR addendum for the two mechanism choices (init-container VG provisioning + boot-disk-partition) that ADR-009 left unspecified.

This is a **Talos-coupled** feature (ADR-012): enabling it must render BOTH the Flux catalog entry AND the Talos patches, so it needs render → bootstrap-talos → flux reconcile, not a pure-git toggle.

**Agents must NOT run live cluster operations** (no `talosctl apply`, no `flux reconcile`, no `kubectl`). They render, lint, build, and review only. The live-apply sequence (§7) is for the USER.

---

## 1a. Resolved decisions (AUTHORITATIVE — override anything below that conflicts)

User-confirmed 2026-06-29, after the plan was drafted. Where these conflict with §2–§8, THESE WIN.

1. **Namespace = `csi-openebs-system`** (NOT `openebs-system`). New repo convention: **CSI drivers get a `csi-<name>-system` namespace** to group all storage CSI by purpose (a future Hetzner CSI → `csi-hcloud-system`). Override the chart's default namespace via `targetNamespace: csi-openebs-system` + `install.createNamespace`. Resolves Q1. Record this convention in the ADR addendum + `_components/README.md`.
2. **Partition size = exactly 20GB** per node. Use `minSize: 20GB` AND `maxSize: 20GB` (deterministic fixed-size partition; no `grow`). Resolves Q2.
3. **Thin provisioning ENABLED.** StorageClass carries `parameters.thinProvision: "yes"`. Keep `dm_thin_pool` in the kernel-module patch (already planned) and enable the snapshot CRDs/snapshot-class as the chart requires for thin/snapshots. Document the talos#9300 boot-loop caveat for thin LVs in the runbook. Resolves Q3.
4. **`vgName` is a Flux-side concern, hardcoded in the catalog.** The StorageClass `volgroup` and the init-container `vgcreate <vgName>` both live in the Flux catalog, so they are inherently consistent and the user overrides the vgName via a **Flux overlay** when needed — NOT via Ansible/config. Therefore:
   - **DROP `openebs.vgName` from the `storage.volumes[]` schema.** The Ansible/config side owns ONLY: the on/off toggle (`features.openebs`) and the Talos volume `name`/`type`/`diskSelector`/`minSize`/`maxSize`.
   - Mark the OpenEBS-backing volume with a simple boolean **`openebs: true`** on the volume item (drives the `openebs ⇒ raw` validation), not a vgName object.
   - The remaining Talos↔Flux coupling is the volume **`name`** (config) ↔ the init-container **partlabel** `r-<name>` (hardcoded in the catalog). That single coupling is documented + reviewer-checked. Resolves R3 (the coupling is now `name`, not `vgName`).
   - Pin a concrete vgName in the catalog (e.g. `vgName: openebs-vg`) used by both the StorageClass `volgroup` and the init container.

---

## 2. Research findings (confirmed / decided during planning)

### 2.1 Talos `RawVolumeConfig` (v1.13) — CONFIRMED
- It is a **separate machine-config document** (`apiVersion: v1alpha1`, `kind: RawVolumeConfig`), NOT a strategic merge into `machine:`. Shape:
  ```yaml
  apiVersion: v1alpha1
  kind: RawVolumeConfig
  name: openebs-lvm                # 1–34 chars, ASCII letters/digits/'-'
  provisioning:
    diskSelector:
      match: system_disk           # CEL expression (see below)
    minSize: 20GB
    maxSize: 30GB
    # grow: false                  # do NOT grow to fill the disk — leave OS headroom
  ```
- The partition surfaces at the stable symlink **`/dev/disk/by-partlabel/r-<name>`** (label is auto-derived as `r-<name>`). For `name: openebs-lvm` → `/dev/disk/by-partlabel/r-openebs-lvm`.
- **System-disk diskSelector expression — CONFIRMED:** the docs example uses `match: "!system_disk"` for *non*-system disks. The boolean CEL variable is **`system_disk`**, so to target the boot/system disk itself the expression is **`match: system_disk`**. Talos provisions the raw partition from free space on that disk. (Source: docs.siderolabs.com/talos/v1.13 raw-volumes + RawVolumeConfig reference.)
- `minSize`/`maxSize` accept human-readable byte sizes (`20GB`, `30GiB`). The boot disk is ~80 GB; recommend default **`minSize: 20GB` / `maxSize: 30GB`** (tunable via `storage.volumes`), leaving room for the Talos OS + ephemeral.
- **No LUKS** (locked decision #3): the schema deliberately omits an `encryption` field so plain `RawVolumeConfig` is the only shape that can render (talos#13354 LUKS+LVM shutdown/upgrade deadlock).

### 2.2 Multi-doc patch application — CONFIRMED PATH, verify in implementation
The existing `render_talos.yml` + `talos_config` already pass multi-document patches via `talosctl gen config --config-patch @file`, and `all/90-user-raw.yaml.j2` explicitly documents that **separate config documents** (e.g. `HostnameConfig`) in a patch file are supported. A `RawVolumeConfig` document is applied the same way — it rides as an additional document in an all-node `--config-patch`. **Implementor must verify** the rendered RawVolumeConfig survives `gen config --config-patch` (vs. needing a `talosctl patch mc` step); the 90-user-raw precedent says it does. Residual-risk R1.

### 2.3 OpenEBS `lvm-localpv` chart — CONFIRMED
- Use the **standalone `lvm-localpv` chart** (repo `https://openebs.github.io/lvm-localpv`), NOT the umbrella `openebs` chart (umbrella pulls hostpath + zfs + mayastor we don't want).
- **No native `initContainers` values hook on the node DaemonSet.** Inspected `deploy/helm/charts/values.yaml` (`develop`): `lvmNode` exposes `tolerations`, `nodeSelector`, `securityContext`, `podLabels`, `annotations`, `resources`, `priorityClass`, `kubeletDir` — but **no `initContainers`**. Docker Hardened-Images guide confirms "the upstream Helm chart does not currently support initContainers." → **Decision: inject the VG-provisioning init container via a Flux HelmRelease `spec.postRenderers` kustomize strategic-merge patch** onto the `openebs-lvm-localpv-node` DaemonSet. (Tolerations CAN go through chart values — `lvmNode.tolerations` exists.)
- Node DaemonSet component name pattern: `<release>-lvm-localpv-node`; controller Deployment `<release>-lvm-localpv-controller`. With releaseName `openebs-lvm-localpv` the DaemonSet is `openebs-lvm-localpv-node`. **Implementor must confirm the exact rendered DaemonSet name via `helm template`** before writing the postRenderer `target.name` (it depends on the fullname template + releaseName). Residual-risk R2.
- **Chart version:** standalone `lvm-localpv` released `v1.8.1` (2026-02) and `v1.9.0` (changelog dated 2026-05-21); `develop` ships `lvmPlugin` image tag `1.10.0-develop`. **Pin to the latest STABLE standalone chart release at implementation time** (expected `1.9.0`; fall back to `1.8.1` if `1.9.0` is not yet a tagged stable release). No floating tags. k8s-implementor confirms the exact latest stable via `helm search repo`/releases and pins both the chart version and the `lvmPlugin.image.tag` (pin the app image too — do not inherit a `-develop` default).
- StorageClass (ADR-009 / locked decisions): name **`openebs-lvm-localpv`**, `provisioner: local.csi.openebs.io`, `parameters.storage: "lvm"`, `parameters.volgroup: <vgName>`, `volumeBindingMode: WaitForFirstConsumer`, `allowVolumeExpansion: true`.
- **Not pre-Flux**: purely Flux-managed. No `flux_bootstrap` role changes, no Ansible Helm install, no adoption handshake (unlike cilium/ccm). The catalog entry stands alone.

### 2.4 Device-mapper kernel modules — CONFIRMED REQUIRED
- lvm-localpv prereqs: **`dm-snapshot` (dm_snapshot) is mandatory**; **`dm_thin_pool` required for thin-provisioning/snapshots**; `dm_mod` is the base device-mapper module.
- Talos does NOT guarantee these are loaded; they are loaded declaratively via **`machine.kernel.modules`**:
  ```yaml
  machine:
    kernel:
      modules:
        - name: dm_mod
        - name: dm_snapshot
        - name: dm_thin_pool
  ```
- This is a SEPARATE generated all-node Talos patch fragment, gated on `features.openebs`. (Sources: lvm-localpv quickstart prereqs; openebs thin-provisioning/snapshot docs; Talos kernel-module machine-config docs; talos#7483, talos#9300.)
- **Boot caveat for the runbook (talos#9300):** active thin/raid LVs + the kernel-module load race can cause Talos boot loops on reboot/upgrade even WITHOUT LUKS. We are not using raid LVs; thin pools only if a thin StorageClass is added later. Document the caveat and the recovery (wipe/re-provision).

### 2.5 No deviation from ADR-009
ADR-009 (Accepted) already names: `RawVolumeConfig` raw device, `diskSelector`, `storage.volumes`, render-both-halves, StorageClass `openebs-lvm-localpv`, two-tier storage. This plan is consistent. The two things ADR-009 left UNSPECIFIED — (a) *how* the VG gets created (init container) and (b) *which device* (partition the boot disk vs. dedicated disk) — warrant a short **ADR-009 addendum**, not a new ADR (no decision reversed).

---

## 3. Proposed `storage.volumes[]` schema shape (locked design input for Step 1)

Minimal but future-proof. Structured item schema in `config/schemas/cluster-input.schema.json`:

> Per §1a decision #4, `vgName` is NOT in this schema — it is hardcoded Flux-side. The backing volume is marked with a boolean `openebs: true`.

```yaml
storage:
  volumes:
    - name: openebs-lvm        # → Talos volume name; partlabel /dev/disk/by-partlabel/r-openebs-lvm
      type: raw                # enum: [raw]  (only raw supported; UserVolume rejected for openebs)
      diskSelector: system_disk  # CEL match expression; default targets the boot/system disk
      minSize: 20GB
      maxSize: 20GB            # fixed 20GB (§1a #2): minSize == maxSize, no grow
      openebs: true            # marks this volume as the OpenEBS VG backing device
```

JSON-Schema item (`storage.volumes.items`):
- `name`: `string`, `pattern ^[A-Za-z0-9-]{1,34}$` (Talos volume-name rule).
- `type`: `enum ["raw"]` (extension point: future `user`).
- `diskSelector`: `string` (CEL); default/recommended `system_disk`.
- `minSize`, `maxSize`: `string` (human byte size).
- `openebs`: `boolean` — `true` marks this volume as the OpenEBS VG backing device. (vgName lives Flux-side, §1a #4.)
- `additionalProperties: false` on the item — **structurally rejects any `encryption` key** (enforces no-LUKS without extra logic).
- Replace the current `"volumes": { "type": "array" }` with `"volumes": { "type": "array", "items": { …above… } }`.

`config/defaults/cluster.yaml`: keep `features.openebs: false`, `storage.volumes: []` (unchanged). `config/clusters/kube1/cluster.yaml`: flip `features.openebs: true` and add ONE volume entry (the example above) — this is the only user-owned enablement point.

---

## 4. Compute-layer wiring decision (for Step 4)

`render_flux.yml` currently maps ONLY from `config_render_bootstrap_pre_flux_components` (pre-Flux Helm releases). OpenEBS is **Flux-managed/post-bootstrap**, so it must NOT enter that list (would wrongly imply an Ansible pre-Flux install).

**Decision:** introduce a SEPARATE computed list `config_render_flux_features` for post-bootstrap Flux-only catalog entries, and have `render_flux.yml` build `config_render_flux_selected_resources` from BOTH lists (`config_render_bootstrap_pre_flux_components` → cilium/hcloud-ccm, AND `config_render_flux_features` → openebs/+future cert-manager/ingress/hcloud-csi). This keeps the pre-Flux vs Flux-managed distinction explicit and is the extension point for the other still-deferred features.

Also compute:
- `config_render_storage_raw_volumes`: the list of `storage.volumes` entries with `type == 'raw'` (drives the RawVolumeConfig render; today == all volumes).
- `config_render_openebs_enabled`: convenience bool = `features.openebs` (gates kernel-module patch + openebs in `config_render_flux_features`).
- `config_render_openebs_volume` / `config_render_openebs_vgname`: the single volume whose `openebs.vgName` is set (drives nothing in Ansible directly, but documents the Talos↔Flux coupling; the vgName is hand-kept in lock-step in the catalog StorageClass — see R3).

---

## 5. Ordered, dependency-aware steps

Phases A (Talos/Ansible side) and B (Flux side) are largely independent and can run in parallel after Step 0; Step 9 (render verify) needs both. Review (C) follows; docs/ADR (D); report (E) last.

### Phase A — render-compiler + Talos (agent: `ansible-implementor`)

**Step 1 — Schema: structured `storage.volumes[]`** · `ansible-implementor`
- File: `config/schemas/cluster-input.schema.json`. Replace `storage.volumes` with the structured item schema from §3. Leave `features.openebs` bool as-is.
- Verify: schema is valid JSON; `config/defaults/cluster.yaml` (empty list) and the kube1 config (Step 2) still validate against it (yaml-language-server `$schema` reference resolves). No render run needed yet.

**Step 2 — Config inputs** · `ansible-implementor`
- `config/defaults/cluster.yaml`: keep `features.openebs: false`, `storage.volumes: []` (no change beyond a clarifying comment that the structured shape is now schema-enforced).
- `config/clusters/kube1/cluster.yaml`: set `features.openebs: true`; add the single `storage.volumes` entry (name `openebs-lvm`, type `raw`, diskSelector `system_disk`, minSize `20GB`, maxSize `30GB`, `openebs.vgName: openebs-vg`). Comment the boot-disk-partition rationale + headroom + talos#13354 (no LUKS) inline.
- Verify: deferred; validated end-to-end in Step 9.

**Step 3 — Validation rewrite** · `ansible-implementor`
- File: `infra/ansible/roles/config_render/tasks/validate.yml`.
- **Replace** the `storage.volumes` deferred-guard (lines ~242–252) with real validation:
  - Each volume: `name` matches the Talos pattern + is unique across the list; `type == 'raw'`; `minSize`/`maxSize` present and non-empty.
  - Reject any volume carrying an `encryption`/`luks` key (belt-and-suspenders on top of schema `additionalProperties:false`) — cite talos#13354.
  - Reject `openebs.vgName` set on a `type != 'raw'` volume.
  - **ADR-013 `openebs ⇒ raw` rule:** if `features.openebs: true`, assert there is exactly ONE volume with `type == 'raw'` AND `openebs.vgName` non-empty (a backing raw volume must exist; not a UserVolume). Clear fail_msg pointing at `storage.volumes`.
  - (Optional) if `features.openebs: false` but a volume defines `openebs.vgName`, warn/allow (provisions the partition but no consumer) — recommend a soft `debug` warn, not a hard fail.
- **KEEP** the `talos.install.disk` and `talos.install.diskSelector` deferred guards UNCHANGED — this feature uses `storage.volumes`/RawVolumeConfig, NOT `install.diskSelector`.
- **Do NOT touch** the `features.hcloudCsi` provider-gate, the CCM tri-state logic, or any flux-auth checks.
- Verify: `ansible-lint`. Functional validation in Step 9.

**Step 4 — Compute** · `ansible-implementor`
- File: `infra/ansible/roles/config_render/tasks/compute.yml`.
- Add `config_render_flux_features` (post-bootstrap Flux-only catalog list): `['openebs'] if features.openebs else []` (extension point for future cert-manager/ingress/hcloud-csi). Mirror the comment style of the existing `config_render_talos_components` block.
- Add `config_render_storage_raw_volumes` (filter `storage.volumes` to `type == 'raw'`) and `config_render_openebs_enabled`.
- Do NOT add openebs to `config_render_bootstrap_pre_flux_components` (not pre-Flux). Do NOT add a Talos `config/talos/components/*.yaml` entry — the openebs Talos artifacts are COMPUTED fragments (RawVolumeConfig + kernel modules), not a verbatim component source, so they are real `.j2` templates (Step 5), not component copies.
- Verify: `ansible-lint`; values confirmed in Step 9 render.

**Step 5 — Talos render: RawVolumeConfig + kernel modules** · `ansible-implementor`
- New templates under `infra/ansible/roles/config_render/templates/talos_patches/all/`:
  - `25-kernel-modules.yaml.j2` — `machine.kernel.modules: [dm_mod, dm_snapshot, dm_thin_pool]` (real template; strategic merge into the v1alpha1 doc). Numeric prefix 25: after `20-install`, a computed integration fragment.
  - `30-raw-volumes.yaml.j2` — one `RawVolumeConfig` document per `config_render_storage_raw_volumes` entry (separate docs, `---` separated). Real template (computed from structured fields) — NOT `{% raw %}`-wrapped (that wrapping is only for verbatim human-authored source manifests; these are computed fragments like `20-install.yaml.j2`). Emit `name`, `provisioning.diskSelector.match`, `minSize`, `maxSize`. Add a **fail-loud guard comment** + design so the init container (Phase B) — not Talos — is what verifies the partlabel exists; Talos itself only provisions on matching free space.
- File: `infra/ansible/roles/config_render/tasks/render_talos.yml`. Add paired copy/remove tasks for BOTH files, gated on `config_render_openebs_enabled` (render when enabled; `state: absent` when disabled) — follow the exact pattern of the `10-external-cloud-provider` and `90-user-raw` paired tasks so toggling `features.openebs` off cleanly removes the generated files and check-mode diffs clean.
- Both files land in `generated/all/` (RawVolumeConfig + kernel modules apply to every node; on kube1 all 3 hybrids are identical so all match the system-disk selector).
- Confirm `talos_config` consumes them automatically: it discovers `generated/all/*.yaml.j2` by glob and renders each (`main.yml` lines ~31–62) — no `talos_config` change needed. Update the patch-ordering comment block in `talos_config/tasks/main.yml` (~lines 290–301) to list the two new all-node files.
- Verify: `ansible-lint`. Rendered output checked in Step 9.

### Phase B — Flux catalog (agent: `k8s-implementor`)

**Step 6 — Flux render wiring** · `ansible-implementor` (render-role file, Ansible lane) ⚠️ ownership note
- File: `infra/ansible/roles/config_render/tasks/render_flux.yml` + `templates/flux/selected/kustomization.yaml.j2`.
- Extend `config_render_flux_selected_resources` to also append `../../_components/openebs` when `'openebs' in config_render_flux_features`. The `.j2` template already joins the list generically — likely no template change, just the resources computation in `render_flux.yml`.
- This is a `config_render` (Ansible-lane) file, so it belongs to `ansible-implementor`, but it is logically the bridge to Phase B. Sequence it after Step 4 (needs `config_render_flux_features`).
- Verify: `ansible-lint`; rendered `generated/selected/kustomization.yaml` lists openebs in Step 9.

**Step 7 — OpenEBS catalog base** · `k8s-implementor`
- New dir `flux/infrastructure/controllers/_components/openebs/` (per `_components/README.md` contract — ordinary kustomize base, NOT a `kind: Component`):
  - `namespace.yaml` — `Namespace` **`openebs-system`** (repo dedicated-`<component>-system` convention; chart default `openebs` is overridden via `targetNamespace`). Confirm naming with user (open question Q1).
  - `helmrepository.yaml` — `source.toolkit.fluxcd.io/v1 HelmRepository`, `url: https://openebs.github.io/lvm-localpv`, `interval: 1h`, in `openebs-system`.
  - `helmrelease.yaml` — `helm.toolkit.fluxcd.io/v2 HelmRelease`, name/releaseName `openebs-lvm-localpv`, `targetNamespace: openebs-system`, `install.createNamespace: true`, chart `lvm-localpv` pinned (latest stable, §2.3), `interval: 1h`. **No `dependsOn`** unless a dependency on storage CRDs/snapshot-controller is identified (the chart bundles its own CRDs via `crds.*` values). Values:
    - `lvmNode.tolerations`: full list to schedule on kube1 hybrids — include the control-plane taint keys AND **`node.cloudprovider.kubernetes.io/uninitialized`** (same external-cloud-provider concern Cilium hit, issue #41921). Re-state any chart defaults since tolerations replace, not merge.
    - Pin `lvmPlugin.image.tag` to the matching stable app version (no `-develop`).
    - Keep `crds.lvmLocalPv.enabled: true`; decide on `crds.csi.volumeSnapshots` (enable only if snapshots wanted; otherwise leave default — verify no conflict with an existing snapshot-controller).
    - `spec.postRenderers`: kustomize `patches` strategic-merge injecting the **privileged VG-provisioning initContainer** onto the node DaemonSet (`target: { kind: DaemonSet, name: <confirmed-rendered-name> }`). Init container:
      - privileged (`securityContext.privileged: true`), mounts host `/dev` (hostPath), and a host mount for LVM (`/run/lock`/`/etc/lvm` as needed).
      - command: idempotent guard — `vgs <vgName>` succeeds → exit 0; else verify `/dev/disk/by-partlabel/r-<name>` EXISTS and **fail loudly if absent** (never `pvcreate`/`vgcreate` a wrong/blank device), then `pvcreate /dev/disk/by-partlabel/r-<name>` + `vgcreate <vgName> /dev/disk/by-partlabel/r-<name>`.
      - image: a small image that ships LVM2 userspace; prefer reusing the `openebs/lvm-driver` image (already pulled, ships lvm2) to avoid a second pinned image — confirm it has `vgs`/`pvcreate`/`vgcreate` (it does; it's the lvm-localpv plugin). Pin the tag.
  - StorageClass `openebs-lvm-localpv.yaml` — `storage.k8s.io/v1 StorageClass`, name `openebs-lvm-localpv`, `provisioner: local.csi.openebs.io`, `parameters: { storage: "lvm", volgroup: "openebs-vg" }`, `volumeBindingMode: WaitForFirstConsumer`, `allowVolumeExpansion: true`. (volgroup MUST equal the `storage.volumes[].openebs.vgName` from Step 2 — see R3.)
  - `kustomization.yaml` — ordinary base: `resources: [namespace.yaml, helmrepository.yaml, helmrelease.yaml, openebs-lvm-localpv.yaml]`.
- Update `_components/README.md` "Current state" to add the openebs entry (template-maintainer-owned doc edit, same lane).
- Verify (k8s-implementor lint gateway): `kustomize build flux/infrastructure/controllers/_components/openebs`, `flux build`/`helm template` of the HelmRelease values, `kubeconform`/`yamllint`. **Run `helm template lvm-localpv` to capture the exact node DaemonSet name** for the postRenderer target (R2).

### Phase C — Review

**Step 8a — Ansible review** · `ansible-reviewer`
- Audit Steps 1–6 (schema, configs, validate, compute, render_talos templates, render_flux): idempotency, paired copy/remove + check-mode cleanliness, FQCN, var namespacing (`config_render_*`), the kept `install.disk`/`diskSelector` guards, the `openebs ⇒ raw` rule, no regression to hcloudCsi/CCM/flux-auth logic. Runs `ansible-lint` + `render-config.yml --check --diff` reasoning. Loop fixes back to `ansible-implementor`.

**Step 8b — Flux review** · `k8s-reviewer`
- Audit Step 7: versions pinned (chart + all images, no `:latest`/`-develop`), tolerations include the uninitialized key, postRenderer target name correct, StorageClass volgroup == vgName, namespace convention, no accidental `dependsOn`/CNI-ordering issues, catalog-contract compliance (ordinary base). Loop fixes back to `k8s-implementor`.

**Step 9 — Full render verification** · `ansible-implementor` (after 8a fixes)
- Run `ansible-playbook playbooks/render-config.yml` and `-e config_render_check=true` (check mode) and confirm:
  - `generated/all/25-kernel-modules.yaml.j2` and `30-raw-volumes.yaml.j2` are produced with correct content; absent when `features.openebs:false` (toggle test).
  - `generated/selected/kustomization.yaml` lists `../../_components/openebs` alongside cilium + hcloud-ccm.
  - Check mode diffs CLEAN (no phantom drift; worker `.gitkeep` sentinel untouched).
- Re-run `kustomize build` on `generated/selected` (now referencing openebs) to confirm the whole controllers overlay still builds.

### Phase D — Docs & ADR

**Step 10 — Runbook** · `docs-writer`
- New `docs/` runbook (e.g. `docs/openebs-local-storage.md` or under `docs/kube1/`): how to enable (`features.openebs: true` + `storage.volumes` entry → render → bootstrap-talos → flux reconcile), how to resize the partition (requires wipe/re-provision — ADR-009 consequence), the reboot/upgrade caveat (talos#13354 + talos#9300: active LVs can block shutdown/boot even without LUKS), VG/init-container recovery, and the two-StorageClass choice (`hcloud-volumes` vs `openebs-lvm-localpv`). Operational only — no rationale/ADR content.

**Step 11 — ADR addendum + CLAUDE.md** · `adr-writer` (ADR) + note for USER (CLAUDE.md)
- `adr-writer`: append an addendum to `docs/cluster-bootstrap/adrs/009-openebs-local-lv-lvm.md` recording the two mechanism decisions ADR-009 left open: (a) **VG provisioning via a privileged idempotent initContainer on the node DaemonSet, injected by a Flux HelmRelease postRenderer** (chart has no initContainers hook), and (b) **partition the single boot/system disk via `RawVolumeConfig` `diskSelector: system_disk`** (kube1 nodes have one NVMe disk). Note the device-mapper kernel-module requirement. No decision reversed → addendum, not a new ADR.
- CLAUDE.md "Implementation status" + Architecture notes update: the USER owns the one-line Architecture table index; flag the suggested status edit (move OpenEBS from "Still deferred" to "Working — built/reviewed, live-unverified") for the user to apply, or have `docs-writer` propose the diff. Do NOT silently rewrite the user-owned index.

### Phase E — Report

**Step 12 — Closing report** · `report-writer` (call LAST)
- Write `.ai/reports/2026-06-29-openebs-lvm-csi.md`: what changed across `config/`, `infra/ansible/`, `flux/`, `docs/`; what was validated (lint, render check-mode, kustomize/helm build, reviews); the residual risks below; and the live-apply steps the user must run.

---

## 6. Cross-cutting constraints checklist (every implementor step must honor)
- **Paired copy/remove** for each new generated file; toggling `features.openebs: false` removes all openebs artifacts; check-mode (`config_render_check=true`) diffs clean.
- **Pin everything**: Talos stays v1.13.3; helm chart pinned; all images pinned (driver image, init-container image, csi sidecars inherit chart-pinned tags). No `:latest`, no `-develop`.
- **Idempotency**: init container guarded by `vgs <vgName>`; render tasks idempotent; second render is a no-op.
- **Fail loud**: init container aborts if `/dev/disk/by-partlabel/r-<name>` is missing (never touches the wrong disk).
- **Lint gateways**: ansible-lint + `--check --diff` (Ansible lane); `kustomize build` + `flux build`/`helm template` + `kubeconform`/`yamllint` (Flux lane) BEFORE handoff to reviewers.

## 7. Live-apply sequence (USER runs — agents must NOT)
1. `ansible-playbook playbooks/render-config.yml` — regenerate group_vars, Talos patches, selected kustomization; review git diff.
2. `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml` — applies the RawVolumeConfig (carves the `r-openebs-lvm` partition) + the device-mapper kernel-module patch. **May require a reboot** for module load / partition provisioning; if Talos reports a reboot is needed, run `upgrade-talos.yml` or reboot the node. Verify with `talosctl get volumestatus r-openebs-lvm` and `talosctl read /proc/modules | grep dm_`.
3. Flux reconcile (`flux reconcile kustomization flux-system --with-source`) — Flux deploys the openebs catalog; the node-DaemonSet initContainer creates the PV+VG on each node; confirm `kubectl get sc openebs-lvm-localpv` and node pods Ready.

## 8. Residual risks & open questions (USER to decide before implementation)
- **R1 (verify):** `RawVolumeConfig` as an additional document via `gen config --config-patch` — strongly expected to work (90-user-raw HostnameConfig precedent), but the implementor must confirm during render/verify; fallback is a `talosctl patch mc` step in `talos_config` (larger change).
- **R2 (verify):** exact rendered node-DaemonSet name for the postRenderer `target.name` — confirm via `helm template` against the pinned chart + releaseName before writing the patch.
- **R3 (coupling):** the StorageClass `parameters.volgroup` and `storage.volumes[].openebs.vgName` are hand-kept in lock-step (Talos↔Flux coupling point, ADR-013). The catalog StorageClass is template-owned and the vgName is in user-owned kube1 config — a mismatch silently breaks provisioning. Mitigation options: (a) document the contract + reviewer check (planned), or (b) future enhancement to render the StorageClass volgroup from config. Decide whether (b) is in-scope now or deferred.
- **Q1 (decide):** namespace name — `openebs-system` (repo dedicated-`<component>-system` convention) vs the chart default `openebs`. Plan assumes `openebs-system`; confirm.
- **Q2 (decide):** partition size default — plan proposes `minSize 20GB / maxSize 30GB` on the ~80 GB boot disk. Confirm or tune.
- **Q3 (decide):** thin-provisioning — include `dm_thin_pool` + snapshot CRDs now (planned: yes, modules loaded; snapshot CRDs optional) or keep thick-only to minimize the talos#9300 boot-loop surface. Decide whether to ship a thin StorageClass.
- **Q4 (verify at impl time):** exact latest STABLE standalone `lvm-localpv` chart version (`1.9.0` expected, `1.8.1` fallback) and matching `lvmPlugin` app image tag.
- **Live-unverified:** per CLAUDE.md, this plan delivers built+reviewed+validated code, not observed-running behavior.
