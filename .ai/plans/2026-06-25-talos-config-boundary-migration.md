# Plan: Migrate kube1 to ADR-016 Talos configuration boundary

Date: 2026-06-25

## Goal and scope

Migrate kube1 from the ADR-013 “structured helpers + raw fragments” Talos model to the accepted ADR-016 boundary:

- Human-authored Talos node configuration lives as Talos-shaped machine configuration patch manifests.
- Ansible remains the compiler/orchestrator: it loads defaults and a selected cluster, chooses shared base/components and cluster overlays, renders only computed integration fragments, and keeps generated outputs where existing roles need them.
- Current kube1 behavior remains compatible during migration: `render-config.yml` should still produce an equivalent `infra/talos/patches/generated/` stack before any consuming role changes are relied on.
- Playbooks should be prepared for `-e config_cluster=kube1`.

Out of scope for this implementation slice:

- Applying changes to live Talos nodes.
- Replacing `interface: eth1` with `deviceSelector` based on a live probe.
- Installing deferred features such as hcloud CSI, cert-manager, OpenEBS, or ingress.
- Committing generated machine configs, Talos secrets, decrypted SOPS data, or provider tokens.

## Current-state facts from repo context

- ADR-016 is accepted at `docs/cluster-bootstrap/adrs/016-talos-config-boundary.md` and supersedes ADR-013. It requires Talos-shaped patch manifests as the human source of truth and allows generated integration patch outputs to continue where current roles need them.
- Current config inputs are:
  - `config/defaults/cluster.yaml`
  - `config/overrides/cluster.yaml`
  - `config/overrides/nodes.yaml`
- Current schema still exposes broad Talos helper fields in `config/schemas/cluster-input.schema.json`: `talos.version`, `talos.extensions`, `talos.install`, `talos.network.defaultInterface`, and `talos.patches.*`. `config/schemas/nodes.schema.json` still allows per-node `network` and `patch` fields.
- `config/overrides/cluster.yaml` currently sets kube1’s provider/features and Talos helper values, including `talos.network.defaultInterface.interface: eth1`, `cni.name: cilium`, and `cloud_controller_manager.enabled: auto`.
- `config/overrides/nodes.yaml` is currently `nodes: []`; per-node raw patches are not in use.
- `infra/ansible/playbooks/render-config.yml` is localhost-only and documented as not contacting Hetzner, Kubernetes, Talos APIs, or Flux.
- `infra/ansible/roles/config_render/tasks/load.yml` currently loads defaults + overrides + nodes from fixed paths and deep-merges defaults with overrides.
- `config_render` currently renders generated Talos patch templates under `infra/talos/patches/generated/`:
  - `all/00-install.yaml.j2`
  - `all/10-cloud-provider.yaml.j2` when hcloud CCM is selected
  - optional `all/90-user-raw.yaml.j2`
  - `controlplane/00-network.yaml.j2`
  - `controlplane/10-cni-proxy.yaml.j2` when Cilium is selected
  - optional `controlplane/90-user-raw.yaml.j2`
  - optional `nodes/<node>.yaml.j2`
- `infra/ansible/roles/talos_config/tasks/main.yml` consumes only the generated compatibility tree today. It discovers `generated/all/*.yaml.j2` and `generated/controlplane/*.yaml.j2`, renders them into the gitignored secrets directory, then passes them to `talosctl gen config` as ordered `--config-patch` and `--config-patch-control-plane` flags.
- `talos_config` currently applies per-node raw patches at `apply-config` time, then applies role-required identity patches last: `HostnameConfig`, Kubernetes role labels, and `allowSchedulingOnControlPlanes` for hybrid nodes.
- `talos_config` tracks a generation signature from `talosctl gen config` argv plus rendered patch checksums and removes stale generated machine configs when inputs change. It deliberately does not rotate `secrets.yaml`.

## Official docs to consult before implementation

Implementors/reviewers must keep these official Talos docs open and cite them in comments when behavior is non-obvious:

- Talos configuration patching: <https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/system-configuration/patching>
- Talos reproducible machine configuration: <https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/system-configuration/reproducible-machine-configuration>
- `talosctl apply-config` CLI reference: <https://docs.siderolabs.com/talos/v1.13/reference/cli/#talosctl-apply-config>
- Talos configuration overview / multi-document configuration: <https://docs.siderolabs.com/talos/v1.13/reference/configuration/overview>

Docs facts that affect the implementation:

- `talosctl gen config` accepts ordered repeated `--config-patch`, `--config-patch-control-plane`, and `--config-patch-worker` flags.
- Patch format is auto-detected; strategic-merge patches support multi-document YAML.
- Multiple patches are applied in flag order; later patches can override earlier patches.
- Strategic merge recursively merges maps, replaces scalars, and generally appends lists except documented special cases.
- `machine.network.interfaces` is a keyed list: entries merge when `interface:` or `deviceSelector:` matches. Mixing ownership of the same physical NIC across layers with different merge keys can append duplicate entries.
- A single multi-document patch file cannot modify the same document more than once.
- Generated machine configs should be regenerated from committed inputs and not committed.

## Compatibility and migration strategy

Use a staged compatibility migration:

1. Introduce the new source layout and cluster selector while preserving legacy input paths as a fallback.
2. Make `config/clusters/kube1/cluster.yaml` and `nodes.yaml` the selected-cluster inputs, initially equivalent to current `config/overrides/*`.
3. Add Talos-shaped patch manifest sources under `config/talos/` and `config/clusters/kube1/talos/`.
4. Update `config_render` to compose the new source patch stack and render/copy it into the existing `infra/talos/patches/generated/` compatibility tree.
5. Only after generated output equivalence is proven, update `talos_config` to understand any new scopes needed by the stack (`worker/`, per-node overlays) while retaining the generated tree contract.
6. Deprecate/remove legacy broad Talos helper fields from config inputs in a final cleanup phase once kube1 renders correctly from Talos-shaped manifests.

Key compatibility invariant: before changing `talos_config` behavior, a fresh render for kube1 must produce the same effective patch content/order as the current tree for installed features (`install`, external cloud provider, VIP/network, Cilium/kube-proxy, and no per-node patches).

## Open questions / decision points before or during implementation

1. **Exact directory names:** ADR-016 says the target layout is illustrative. Recommended for this migration: use the ADR’s names exactly (`config/talos/base`, `config/talos/components`, `config/clusters/kube1`). This minimizes surprise and avoids another durable decision.
2. **Generated patch output as intermediate:** Recommended: keep `infra/talos/patches/generated/` for this slice. It preserves compatibility with `talos_config` and makes generated diff review simple. A later slice can decide whether roles consume source manifests directly.
3. **Selected component order representation:** Decide whether order is encoded by filename prefixes (`00-`, `10-`, `20-`) or by an explicit computed list. Recommended: compute an explicit ordered list in Ansible and render files with stable numeric prefixes in `generated/` for reviewability.
4. **VIP/network ownership and old `talos.network.defaultInterface`:** This is the highest-risk boundary. Options:
   - A. Move the full VIP interface patch to `config/clusters/kube1/talos/controlplane.yaml`, duplicating `cluster.vip` in Talos-shaped YAML.
   - B. Keep `cluster.vip` as the API endpoint source and render a narrow generated integration fragment for the VIP/interface.
   - C. Move only interface selection to a Talos-shaped overlay and inject VIP as a generated fragment.

   Recommended staged path: start with **B** for compatibility, then decide separately whether duplicate VIP in human Talos YAML is acceptable. Do not silently keep or expand `talos.network.defaultInterface` as a broad schema.
5. **Legacy field deprecation:** Decide whether legacy `config/overrides/*` and `talos.install/network/patches` fail immediately or warn for one migration release. Recommended: implement fallback/warnings first, then remove after the new kube1 layout renders cleanly.

If any open question changes ADR-016’s durable boundary, pause implementation and assign `adr-writer` to amend ADR-016 only after the user agrees.

## Ordered implementation phases and small task boundaries

### Phase 0 — Baseline and guardrails

Owner: `ansible-implementor`

Tasks:

0.1. Capture the current generated Talos compatibility tree as the baseline for comparison.
- Files/context: `infra/talos/patches/generated/**`, `infra/ansible/roles/config_render/tasks/render_talos.yml`, current generated output.
- Boundary: no implementation changes except optional temporary notes under `.ai/` if needed.
- Validation: record current file list and patch ordering for the implementor handoff.

0.2. Confirm current render check status before migration.
- Command from `infra/ansible/`: `ansible-playbook playbooks/render-config.yml -e config_render_check=true`.
- Boundary: local render/check only; no inventories, no cluster, no provider.

### Phase 1 — Add new config input layout without switching behavior

Owner: `ansible-implementor`

Tasks:

1.1. Add selected-cluster non-Talos config files.
- Files: `config/clusters/kube1/cluster.yaml`, `config/clusters/kube1/nodes.yaml`.
- Scope: copy/migrate current `config/overrides/cluster.yaml` and `nodes.yaml` into the new selected-cluster location with comments updated for ADR-016 and `config_cluster=kube1`.
- Do not yet remove `config/overrides/*`.

1.2. Add shared Talos source directories with minimal safe placeholders.
- Files: `config/talos/base/all.yaml`, `config/talos/base/controlplane.yaml`, `config/talos/base/worker.yaml`.
- Scope: add empty or comment-only Talos-shaped patch manifests only if the composer handles empty files safely; otherwise add directories with `.gitkeep` and defer content to later tasks.

1.3. Add reusable Talos component manifests for static feature/provider fragments.
- Files: `config/talos/components/external-cloud-provider.yaml`, `config/talos/components/cilium.yaml`; optionally placeholder `hcloud-csi.yaml` only if disabled and not selected.
- Scope: move static content, not computed values:
  - `external-cloud-provider.yaml`: `cluster.externalCloudProvider.enabled: true`.
  - `cilium.yaml`: `cluster.network.cni.name: none` and `cluster.proxy.disabled: true`.
- Do not include kube API cert SANs here if they remain computed from `cluster.certSANs.kubernetes`.

1.4. Add kube1 Talos overlay directories.
- Files: `config/clusters/kube1/talos/all.yaml`, `controlplane.yaml`, `worker.yaml`, `nodes/.gitkeep`.
- Scope: keep overlays empty/comment-only until VIP/network ownership is decided, or add only Talos-shaped fragments whose source of truth is no longer duplicated.

Validation gate after Phase 1:

- `ansible-playbook playbooks/render-config.yml -e config_render_check=true` should still pass because behavior has not switched.
- Git diff should show only new source files and no generated output drift.

### Phase 2 — Add `config_cluster` loader and preserve legacy fallback

Owner: `ansible-implementor`

Tasks:

2.1. Update `config_render` defaults for selected cluster paths.
- Files: `infra/ansible/roles/config_render/defaults/main.yml`.
- Scope: add `config_cluster: kube1` default; derive `config_render_cluster_dir`, `config_render_cluster_config_path`, and `config_render_cluster_nodes_path`.
- Preserve legacy `config/overrides/*` path variables for fallback and compatibility.

2.2. Update `config_render` load logic.
- Files: `infra/ansible/roles/config_render/tasks/load.yml`.
- Scope: load `config/defaults/cluster.yaml` plus selected `config/clusters/{{ config_cluster }}/cluster.yaml` and nodes when present; fall back to `config/overrides/*` with a clear deprecation warning if selected-cluster files are absent.
- Keep deep-merge semantics for non-Talos cluster config.

2.3. Update render playbook usage comments.
- Files: `infra/ansible/playbooks/render-config.yml`.
- Scope: document `-e config_cluster=kube1` and the fallback behavior.

Validation gate after Phase 2:

- `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true` passes.
- `ansible-playbook playbooks/render-config.yml -e config_render_check=true` still passes by default.

### Phase 3 — Compose Talos patch sources into the generated compatibility tree

Owner: `ansible-implementor`

Tasks:

3.1. Compute selected Talos components.
- Files: `infra/ansible/roles/config_render/tasks/compute.yml`.
- Scope: derive an ordered list such as `config_render_talos_components` from `cni.name`, `cloud_controller_manager.enabled`, provider, and features.
- Initial mapping:
  - `cni.name == cilium` selects `cilium`.
  - hcloud CCM selected selects `external-cloud-provider`.
  - `features.hcloudCsi` remains false and must not select `hcloud-csi` unless the feature exists.

3.2. Add a patch-stack composition task with explicit order.
- Files: new or existing tasks under `infra/ansible/roles/config_render/tasks/`, likely `render_talos.yml` plus helper vars.
- Scope: compose patch sources in ADR-016 order:
  1. `config/talos/base/<scope>.yaml`
  2. `config/clusters/{{ config_cluster }}/talos/<scope>.yaml`
  3. selected `config/talos/components/*.yaml`
  4. generated computed integration fragments
  5. `config/clusters/{{ config_cluster }}/talos/nodes/<node>.yaml`
- Boundary: do not change Flux rendering or provider group_vars in this task.

3.3. Render/copy static source manifests to `infra/talos/patches/generated/`.
- Files: `infra/ansible/roles/config_render/tasks/render_talos.yml`.
- Scope: copy/render Talos-shaped `.yaml` or `.yaml.j2` sources into stable generated filenames so `talos_config` can still discover them.
- Preserve ordering with numeric prefixes and comments indicating source file and ownership.
- Empty/comment-only sources should either be skipped or rendered as harmless YAML comments; choose one behavior and test it.

3.4. Keep computed integration fragments minimal and clearly owned.
- Files: current computed templates under `infra/ansible/roles/config_render/templates/talos_patches/**` or renamed equivalents.
- Scope: keep only values Ansible must compute or enforce, such as:
  - installer image placeholder from `talos_schematic` (`machine.install.image`).
  - Talos/kube API SAN lists if still sourced from non-Talos cluster config.
  - VIP/network fragment if open question 4 selects the staged recommended path.
- Remove/reduce templates that are now static source components (`externalCloudProvider`, Cilium) once component rendering is in place.

3.5. Migrate per-node patch source from `nodes.yaml` to `config/clusters/kube1/talos/nodes/*.yaml`.
- Files: `infra/ansible/roles/config_render/tasks/render_talos.yml`, `config/clusters/kube1/talos/nodes/`.
- Scope: support per-node source manifests. Keep legacy `nodes[].patch` fallback with deprecation warning until cleanup.
- Boundary: do not alter role-required identity patches in `talos_config`; those remain generated/applied by the role and run last.

Validation gate after Phase 3:

- Fresh render in-place, then `git diff -- infra/talos/patches/generated` must be reviewed for semantic equivalence to baseline.
- `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true` passes after generated outputs are updated.
- Confirm no generated files are written under `infra/talos/secrets/`.

### Phase 4 — Update `talos_config` consumer for new generated scopes

Owner: `ansible-implementor`

Tasks:

4.1. Add worker-scope generated patch discovery and rendering.
- Files: `infra/ansible/roles/talos_config/defaults/main.yml`, `infra/ansible/roles/talos_config/tasks/main.yml`.
- Scope: add `generated/worker` discovery, render to `rendered-patches/worker-*`, and append ordered `--config-patch-worker @...` flags to `talos_config_gen_argv`.
- Boundary: do not change all/controlplane behavior except necessary refactoring.

4.2. Include worker patches in regeneration signature.
- Files: `infra/ansible/roles/talos_config/tasks/main.yml`.
- Scope: include rendered worker patch files in `talos_config_rendered_patch_paths` and `talos_config_gen_signature`.

4.3. Keep per-node overlay application order safe.
- Files: `infra/ansible/roles/talos_config/tasks/main.yml`.
- Scope: generated per-node overlays still apply before role-required identity patches. Document that role-required identity patches intentionally win over user per-node overlays for hostname, node labels, and hybrid scheduling.

Validation gate after Phase 4:

- `ansible-lint` for changed Ansible files.
- `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true`.
- `ansible-playbook playbooks/bootstrap-talos.yml --syntax-check` with each inventory path if syntax-check can run without contacting nodes:
  - `-i inventories/common -i inventories/providers/hcloud`
  - `-i inventories/common -i inventories/providers/manual`
- Do **not** run `bootstrap-talos.yml --check --diff` unless the reviewer confirms it will not execute `talosctl` live checks; current `talos_config` contains `check_mode: false` live `talosctl` reads.

### Phase 5 — Schema migration and legacy deprecation cleanup

Owner: `ansible-implementor`

Tasks:

5.1. Narrow `cluster-input.schema.json` to non-Talos machine-config concerns.
- Files: `config/schemas/cluster-input.schema.json`.
- Scope: keep Talos release/image contract fields needed by Ansible (`talos.version`, `talos.extensions`) if desired, but remove or deprecate broad machine config helper fields (`talos.install`, `talos.network.defaultInterface`, `talos.patches`).
- Add schema comments/examples indirectly in YAML files, not JSON comments.

5.2. Narrow `nodes.schema.json`.
- Files: `config/schemas/nodes.schema.json`.
- Scope: keep node identity/provider facts (`name`, `role`, provider/network facts as needed), remove or deprecate `patch` as the primary patch source.

5.3. Update config defaults and kube1 cluster config after schema change.
- Files: `config/defaults/cluster.yaml`, `config/clusters/kube1/cluster.yaml`, optionally legacy `config/overrides/*` if retained.
- Scope: remove old `talos.install/network/patches` from active selected-cluster config only after corresponding Talos-shaped manifests or computed fragments own the behavior.

5.4. Add validation errors or warnings for legacy fields.
- Files: `infra/ansible/roles/config_render/tasks/validate.yml`.
- Scope: fail clearly if removed legacy fields are set in selected-cluster config after the cleanup point, or warn during the transition if fallback remains.

Validation gate after Phase 5:

- JSON schema validation path in `config_render` still passes.
- Render check passes for `config_cluster=kube1`.
- Generated group_vars and Flux base either unchanged or intentionally explained in diff.

### Phase 6 — Documentation updates

Owner: `docs-writer` for `docs/` updates; `ansible-implementor` or top-level orchestrator for `CLAUDE.md` if it must be updated.

Tasks:

6.1. Update operational docs that mention config editing.
- Candidate files: `docs/upgrades.md`, `docs/adding-a-provider.md`, any existing config/render docs.
- Scope: explain that Talos node configuration is edited as Talos machine configuration patch manifests under `config/talos/` and `config/clusters/<cluster>/talos/`, while provider/features/versions stay in selected cluster YAML.

6.2. Update repo status/guidance if needed.
- Candidate file: `CLAUDE.md`.
- Scope: replace ADR-013 render-compiler text with ADR-016 status once implementation lands; keep implementation status honest and do not claim live cluster validation.

Validation gate after Phase 6:

- Docs reflect the new commands:
  - `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1`
  - `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true`
- Docs retain safety warnings: generated machine configs/secrets are gitignored and not committed; live playbooks are run by the user.

### Phase 7 — Review, report, and handoff

Owners: `ansible-reviewer`, then `report-writer`

Tasks:

7.1. Ansible review.
- Owner: `ansible-reviewer`.
- Scope: review `config/`, `infra/ansible/`, generated `infra/talos/patches/generated/`, and docs references for idempotency, FQCN usage, variable namespacing, compatibility, and secret safety.
- Must verify no live cluster/provider commands were run.

7.2. Fix loop if needed.
- Owner: matching `ansible-implementor` task scoped to reviewer findings only.

7.3. Closing report.
- Owner: `report-writer`.
- Scope: write `.ai/reports/2026-06-25-talos-config-boundary-migration.md` after implementation and review pass.

## Validation required before handoff

Minimum non-live validation:

```bash
cd infra/ansible
ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true
ansible-lint
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check
```

Additional validation where feasible and non-live:

- Run `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1` to update generated files, then review `git diff` for generated Talos patch equivalence.
- If a local `talosctl` validation command can validate rendered machine configs without contacting nodes and without committing secrets, run it against temporary or gitignored outputs only. If not run, document why.
- Review patch ordering by inspecting rendered/generated filenames and the constructed `talos_config_gen_argv` path order.
- Confirm no files under `infra/talos/secrets/` are staged or committed.
- Confirm no decrypted SOPS content appears in diffs.

Validation that must **not** be run by agents without explicit user approval:

- `ansible-playbook ... provision-infra.yml`
- `ansible-playbook ... bootstrap-talos.yml` without `--syntax-check`
- `ansible-playbook ... bootstrap-flux.yml`
- `talosctl apply-config`, `talosctl upgrade`, `talosctl patch machineconfig`, `kubectl`, `flux reconcile`, `hcloud` provider operations.

## Risks, side effects, and safety notes

- **Machine config secret safety:** `talosctl gen config` outputs and `secrets.yaml` contain cluster CA/client material and must stay under gitignored `infra/talos/secrets/` with restrictive permissions.
- **Network interface merge-key hazard:** Talos merges `machine.network.interfaces` by `interface:` or `deviceSelector:`. Do not split the same NIC across layers using different keys. During migration, avoid having one patch own `interface: eth1` and another patch own a `deviceSelector` that selects the same NIC unless the order and intent are explicit and reviewed.
- **VIP duplication hazard:** If `cluster.vip` remains the source for endpoints while a Talos-shaped overlay also contains the VIP, validation must prevent divergence or the team must accept duplication explicitly.
- **Patch ordering hazard:** Moving content from generated templates to source manifests changes order unless the composer preserves numeric ordering. Review final `--config-patch*` argv order, not just filenames.
- **Empty/multi-doc file behavior:** Comment-only or empty YAML files may be harmless to Ansible but not to `talosctl`; either skip empty files or prove they are safe locally before including them.
- **Per-node identity:** User per-node overlays must not override required hostname, node-role labels, or hybrid scheduling unintentionally. Keep role-required identity patches last.
- **Provider cost boundary:** This plan should not create, delete, resize, or rebuild Hetzner resources. Any hcloud image/provision command is cost-incurring and outside this migration validation.
- **Live-cluster boundary:** Agents do not apply machine configs or query live nodes. The user runs live playbooks after local validation and review.

## ADR needs

- ADR-016 already records the accepted Talos configuration boundary and supersedes ADR-013.
- No new ADR is expected if implementation follows ADR-016 and the recommended staged compatibility path.
- If the team chooses a materially different source-of-truth for VIP/network ownership, generated-output consumption, or component ordering than ADR-016 describes, pause and assign `adr-writer` to update ADR-016 after the decision is agreed.
