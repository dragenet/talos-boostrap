# Plan: Migrate Flux toward ADR-017 shared overlay ownership

Date: 2026-06-25

## Goal and scope

Migrate kube1's Flux tree toward ADR-017's three-tier overlay ownership model while coordinating with the existing ADR-016 Talos migration plan instead of duplicating it.

Target ownership model for Flux:

- **template/shared defaults and components**: reusable component catalog and shared base material under `flux/`;
- **user-global overlay**: repo/user preferences applied to every cluster in this repo;
- **per-cluster overlay**: final cluster-specific overrides for `kube1`.

Important scope boundaries:

- Flux remains self-contained under `flux/`; do **not** move Flux manifests under `config/`.
- Talos source layout changes remain owned by the ADR-016 plan: `.ai/plans/2026-06-25-talos-config-boundary-migration.md`.
- This plan may update `config_render` only where it renders Flux-generated selected bases; it must not re-plan Talos patch composition.
- Do not run live-cluster Flux/Kubernetes operations. No `flux reconcile`, no live `kubectl`, no `bootstrap-flux.yml` execution.
- Do not hand-edit `flux/clusters/<cluster>/flux-system/` bootstrap-owned files casually. If the directory is absent locally, do not create it by hand as part of this migration; `flux bootstrap` owns it.
- Preserve pre-Flux adoption contracts: Cilium and hcloud CCM release names, namespaces, chart versions, Helm values, and ordering must remain equivalent before any path switch.

## Current-state facts from repo context

Repo facts inspected for this plan:

- ADR-017 is accepted at `docs/cluster-bootstrap/adrs/017-shared-overlay-ownership.md`. It records the shared ownership model across Talos and Flux, keeps Flux under `flux/`, and leaves implementation questions open around `_components` vs `components` and the role of `base/`.
- ADR-016 is accepted at `docs/cluster-bootstrap/adrs/016-talos-config-boundary.md`. The existing plan is `.ai/plans/2026-06-25-talos-config-boundary-migration.md`.
- Some ADR-016 prerequisites are already present in the tree:
  - `infra/ansible/roles/config_render/defaults/main.yml` defines `config_cluster: kube1` and selected-cluster paths.
  - `infra/ansible/roles/config_render/tasks/load.yml` loads `config/clusters/{{ config_cluster }}/cluster.yaml` and `nodes.yaml` with legacy `config/overrides/*` fallback.
  - `config/clusters/kube1/{cluster.yaml,nodes.yaml}` exists.
  - `config/talos/base/*`, `config/talos/components/*`, and `config/clusters/kube1/talos/*` exist.
- Current Flux controller layout still follows ADR-010/011/012:
  - `flux/infrastructure/controllers/_components/` is the template-owned catalog.
  - `flux/infrastructure/controllers/base/kustomization.yaml` is render-owned and currently references:
    - `../_components/cilium`
    - `../_components/providers/hcloud/ccm`
  - `flux/infrastructure/controllers/base/cluster-vars.yaml` is generated but not referenced by the base kustomization yet.
  - `flux/infrastructure/controllers/kustomization.yaml` is the user-owned overlay and currently has `resources: [base]` with commented patch guidance.
- Current Flux configs/apps are placeholders:
  - `flux/infrastructure/configs/kustomization.yaml` has `resources: []`.
  - `flux/apps/kustomization.yaml` has `resources: []`.
- Current cluster entrypoints:
  - `flux/clusters/kube1/infrastructure.yaml` defines `infrastructure-controllers` pointing at `./flux/infrastructure/controllers`, then `infrastructure-configs` depending on it and pointing at `./flux/infrastructure/configs`; both set `prune: true` and `wait: true`.
  - `flux/clusters/kube1/apps.yaml` defines `apps` depending on `infrastructure-configs`, pointing at `./flux/apps`, with `prune: true`.
- `config_render` currently renders only the Flux controllers base:
  - `infra/ansible/roles/config_render/tasks/render_flux.yml`
  - `infra/ansible/roles/config_render/templates/flux/base/kustomization.yaml.j2`
  - `infra/ansible/roles/config_render/tasks/check.yml` diffs only `flux/infrastructure/controllers/base/kustomization.yaml` for Flux drift.
- `compute.yml` already computes `config_render_bootstrap_pre_flux_components` from `cni.name`, CCM resolution, provider, and Cilium; Flux render maps that to current catalog paths.

## Official docs to consult before implementation

Implementors and reviewers must keep these official Flux docs open before making or reviewing Flux/Kustomize changes:

- Flux repository structure guide: <https://fluxcd.io/flux/guides/repository-structure/>
- Flux bootstrap generic Git server docs: <https://fluxcd.io/flux/installation/bootstrap/generic-git-server/>
- Flux bootstrap command reference: <https://fluxcd.io/flux/cmd/flux_bootstrap_git/>
- Flux Kustomization docs: <https://fluxcd.io/flux/components/kustomize/kustomizations/>
- Flux Kustomization API reference: <https://fluxcd.io/flux/components/kustomize/api/v1/>

Docs facts that affect implementation:

- Flux's recommended monorepo structure keeps `apps/`, `infrastructure/`, and `clusters/<cluster>/` together in the GitOps tree.
- Cluster directories under `clusters/<cluster>` are bootstrap entrypoints; `flux bootstrap --path=clusters/<cluster>` writes `clusters/<cluster>/flux-system/`.
- Apps/infrastructure separation supports ordered reconciliation.
- `Kustomization.spec.dependsOn` gates reconciliation on dependencies being Ready.
- `wait: true` health-checks reconciled resources and makes downstream dependencies wait for health.
- `prune: true` enables garbage collection and must be treated carefully when changing paths or resource ownership.

## Coordination with the ADR-016 Talos plan

Shared prerequisites from `.ai/plans/2026-06-25-talos-config-boundary-migration.md`:

- `config_cluster=kube1` and selected-cluster loading are useful to both plans. Do not create a second cluster selector for Flux.
- `compute.yml` already derives component selections from the selected cluster config. Flux migration should reuse `config_render_bootstrap_pre_flux_components` and avoid new parallel feature-selection logic.
- The ADR-016 plan owns Talos-shaped machine configuration patch manifests under `config/`; this Flux plan must not move, rename, or reinterpret those files.
- If this plan changes `config_render` defaults/check mode, coordinate with ADR-016 so render output base/temp-dir behavior remains shared and deterministic.

Avoid duplicating ADR-016 work:

- Do not re-plan `config/clusters/kube1/*` creation unless a Flux-specific field is needed.
- Do not re-plan `config/talos/*` or `infra/talos/patches/generated/*` migration.
- Do not change Talos validation, Talos schema narrowing, or `talos_config` consumer behavior in Flux implementor tasks.

Dependency note:

- If the ADR-016 implementation is not fully landed in the branch when this plan is executed, Flux work can still proceed after confirming `config_cluster` loading and `config_render_output_base` temp rendering work. Any missing selected-cluster prerequisite should be completed by the ADR-016 plan's `ansible-implementor`, not by a Flux task.

## Compatibility and migration strategy

Use a staged compatibility migration that proves equivalent rendered manifests before switching live Flux paths.

Recommended safety invariant:

1. Keep the current Flux reconciliation paths working while new layout directories are introduced.
2. Render or create the new selected Flux layer in parallel with the existing render-owned `base/` until equivalence is proven.
3. Build both the current and target kustomization paths locally and compare their rendered manifests.
4. Only then either:
   - keep the existing Flux Kustomization paths (`./flux/infrastructure/controllers`, `./flux/infrastructure/configs`, `./flux/apps`) as compatibility shims that point at per-cluster overlays, or
   - switch `flux/clusters/kube1/*.yaml` paths directly to the per-cluster overlays.

The safer initial path is the shim approach: keep cluster entrypoint paths unchanged and make the top-level `kustomization.yaml` in each domain point to `overlays/clusters/kube1`. That reduces Flux `prune` blast radius because the `Kustomization.spec.path` values do not change in the first migration.

## Open questions / decision points

Do not let implementors silently decide these. Resolve them with the user before or during Phase 1.

1. **`_components` vs `components`:**
   - Option A: keep `_components` for now, minimizing path churn.
   - Option B: rename to `components`, matching ADR-017's illustrative target.
   - Recommended staged path: keep `_components` during equivalence testing; rename to `components` in a small follow-up task only after render paths and local builds pass.
2. **Meaning of `base/`:**
   - Current `controllers/base/` is render-owned selected set.
   - Kustomize/Flux convention often uses `base/` for shared human-authored base.
   - Recommended direction: reserve `base/` for shared human-authored content where needed, and move render-owned selected resources to an explicit path such as `generated/selected/`.
3. **Generated selected path name:**
   - Options: `selected/`, `generated/`, `generated/selected/`, or keep `base/`.
   - Recommended for clarity: `generated/selected/` because it encodes both ownership and purpose, avoids overloading `base`, and remains self-contained under `flux/infrastructure/controllers/`.
4. **Exact user-global overlay path:**
   - ADR-017 illustrates `overlays/user/`.
   - Recommended: use `overlays/user/` for all three Flux domains (`controllers`, `configs`, `apps`) and `overlays/clusters/kube1/` for per-cluster overlays.
5. **Cluster entrypoint switch vs compatibility shim:**
   - Option A: leave `flux/clusters/kube1/*.yaml` paths unchanged and make top-level kustomizations shim to per-cluster overlays.
   - Option B: point cluster entrypoints directly to `.../overlays/clusters/kube1`.
   - Recommended first migration: Option A, then consider Option B only after review if the user wants the cluster entrypoint to expose overlay paths directly.
6. **ADR/documentation needs:**
   - ADR-017 already records the decision. No new ADR is expected if implementation chooses one of ADR-017's named migration paths.
   - If the team chooses a different durable ownership model, generated ownership rule, or Flux root than ADR-017 describes, pause and assign `adr-writer` after the decision is agreed.

## Ordered implementation phases and small task boundaries

### Phase 0 — Baseline and decision checkpoint

Owners: top-level orchestrator for decision checkpoint; `k8s-implementor` for local Flux baseline; `ansible-implementor` only if render baseline is needed.

Tasks:

0.1. Resolve naming decisions before implementation.
- Owner: top-level orchestrator with user.
- Scope: decide `_components` vs `components`, render-owned `base/` vs `generated/selected/`, shim vs direct cluster path switch.
- Output: short decision note in the implementation handoff or plan update; do not create an ADR unless choices depart from ADR-017.

0.2. Capture current Flux build baseline.
- Owner: `k8s-implementor`.
- Files/context: `flux/infrastructure/controllers/**`, `flux/infrastructure/configs/**`, `flux/apps/**`, `flux/clusters/kube1/*.yaml`.
- Boundary: no file changes unless writing temporary comparison notes under `.ai/` is explicitly useful.
- Validation:
  - `kubectl kustomize flux/infrastructure/controllers` or `kustomize build flux/infrastructure/controllers`.
  - `kubectl kustomize flux/infrastructure/configs` or `kustomize build flux/infrastructure/configs`.
  - `kubectl kustomize flux/apps` or `kustomize build flux/apps`.
  - Record current rendered resource identities for Cilium and hcloud CCM.

0.3. Confirm current render-check status.
- Owner: `ansible-implementor`.
- Command from `infra/ansible/`: `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true`.
- Boundary: local render/check only; no inventories, no cluster, no provider.

### Phase 1 — Introduce target overlay skeleton without changing reconciliation output

Owner: `k8s-implementor`.

Split into small, non-conflicting tasks:

1.1. Add controllers overlay skeleton.
- Files: `flux/infrastructure/controllers/overlays/user/`, `flux/infrastructure/controllers/overlays/clusters/kube1/`.
- Scope: add kustomizations and empty patch directories/placeholders as needed.
- Initial resources should reference the current selected layer (`../../base` or the agreed future `../../generated/selected`) without changing rendered output.
- Do not edit HelmRelease values, release names, namespaces, or chart versions.

1.2. Add configs three-tier skeleton.
- Files: `flux/infrastructure/configs/base/`, `flux/infrastructure/configs/overlays/user/`, `flux/infrastructure/configs/overlays/clusters/kube1/`.
- Scope: preserve current empty output (`resources: []`) while making ownership layers explicit.
- Keep any top-level `flux/infrastructure/configs/kustomization.yaml` behavior equivalent until the switch phase.

1.3. Add apps three-tier skeleton.
- Files: `flux/apps/base/`, `flux/apps/overlays/user/`, `flux/apps/overlays/clusters/kube1/`.
- Scope: preserve current empty output (`resources: []`) while making ownership layers explicit.

Validation gate after Phase 1:

- Local kustomize build for current top-level paths still succeeds.
- Local build for each new per-cluster overlay succeeds once its referenced selected/base layer exists.
- Rendered manifests are equivalent to Phase 0 for paths that are already switched into the new skeleton.

### Phase 2 — Make render-owned Flux selection explicit

Owner: `ansible-implementor`.

Tasks:

2.1. Update render output path facts.
- Files: `infra/ansible/roles/config_render/defaults/main.yml`.
- Scope: add explicit vars for the chosen Flux selected output path, e.g. `config_render_output_flux_controllers_selected: .../flux/infrastructure/controllers/generated/selected`.
- Preserve `config_render_output_flux_base` during transition if the old `base/` is still rendered for compatibility.

2.2. Update `render_flux.yml` to render the selected controllers set.
- Files: `infra/ansible/roles/config_render/tasks/render_flux.yml`.
- Scope: render the same selected resources to the new explicit path while optionally continuing to render old `controllers/base/kustomization.yaml` until the switch is complete.
- Keep resource selection driven by `config_render_bootstrap_pre_flux_components`.
- Do not add Flux feature toggles unrelated to current Cilium/hcloud CCM selection.

2.3. Update the Flux selected kustomization template.
- Files: `infra/ansible/roles/config_render/templates/flux/**`.
- Scope: update comments to say render-owned selected set, not shared base, and point to the agreed component catalog path.
- If `_components` is renamed to `components`, update resource paths atomically with the catalog rename phase.

2.4. Update render drift checks.
- Files: `infra/ansible/roles/config_render/tasks/check.yml`.
- Scope: diff the new generated/selected Flux output in check mode. If old `base/` remains rendered during transition, check both or explicitly mark old `base/` deprecated.

Validation gate after Phase 2:

- From `infra/ansible/`: `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true` passes.
- Fresh in-place render updates only expected generated Flux selected files.
- New selected output builds locally and renders the same Cilium/hcloud CCM objects as old `controllers/base`.

### Phase 3 — Catalog path migration, if approved

Owner: `k8s-implementor` for Flux tree rename; `ansible-implementor` for render path update if needed.

This phase is conditional on the Phase 0 decision.

3.1. Rename or duplicate catalog path safely.
- Owner: `k8s-implementor`.
- Files: `flux/infrastructure/controllers/_components/**` and/or `flux/infrastructure/controllers/components/**`.
- Scope: if moving to `components/`, perform a small, isolated rename/copy and update local kustomization references. Do not modify component contents beyond path comments.
- Preserve Cilium and hcloud CCM adoption contract exactly.

3.2. Update render resource mapping.
- Owner: `ansible-implementor`.
- Files: `infra/ansible/roles/config_render/tasks/render_flux.yml`, relevant templates.
- Scope: update `cilium` and `hcloud-ccm` resource paths from `_components` to `components` only when the catalog path exists.

Validation gate after Phase 3:

- Old selected build and new selected build are equivalent except for comments/path-only changes.
- `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true` passes.

### Phase 4 — Switch top-level Flux paths to the per-cluster overlay chain

Owner: `k8s-implementor`.

Tasks:

4.1. Switch controllers top-level kustomization to cluster overlay shim.
- Files: `flux/infrastructure/controllers/kustomization.yaml`.
- Scope: make it reference `overlays/clusters/kube1` instead of directly referencing the render-owned selected set.
- Boundary: do not change `flux/clusters/kube1/infrastructure.yaml` in this task if using the shim strategy.

4.2. Switch configs top-level kustomization to cluster overlay shim.
- Files: `flux/infrastructure/configs/kustomization.yaml`.
- Scope: make it reference `overlays/clusters/kube1` while preserving empty rendered output.

4.3. Switch apps top-level kustomization to cluster overlay shim.
- Files: `flux/apps/kustomization.yaml`.
- Scope: make it reference `overlays/clusters/kube1` while preserving empty rendered output.

4.4. Optional direct cluster entrypoint path switch.
- Files: `flux/clusters/kube1/infrastructure.yaml`, `flux/clusters/kube1/apps.yaml`.
- Scope: only if the user chose direct overlay paths in Phase 0, change paths to the per-cluster overlay directories.
- Must preserve `dependsOn`, `interval`, `retryInterval`, `timeout`, `prune`, and `wait` semantics. If `apps` still lacks `wait`, decide whether adding it is part of this migration or separate cleanup; do not silently change readiness semantics.

Validation gate after Phase 4:

- Build current Flux Kustomization paths:
  - `kubectl kustomize flux/infrastructure/controllers` or `kustomize build flux/infrastructure/controllers`
  - `kubectl kustomize flux/infrastructure/configs` or `kustomize build flux/infrastructure/configs`
  - `kubectl kustomize flux/apps` or `kustomize build flux/apps`
- Build per-cluster overlay paths directly.
- Compare Phase 0 vs Phase 4 controller rendered manifests; differences must be path/comment-only or explicitly explained.
- No live `kubectl` or `flux reconcile` commands.

### Phase 5 — Cleanup deprecated Flux layout names and comments

Owners: `k8s-implementor` for Flux comments/files; `ansible-implementor` for render cleanup.

Tasks:

5.1. Remove old render-owned `controllers/base/` only after replacement is proven.
- Owner: `ansible-implementor`.
- Files: `flux/infrastructure/controllers/base/`, `infra/ansible/roles/config_render/**`.
- Scope: stop rendering old base and update check mode once no kustomization references it.
- Boundary: preserve generated file headers on any generated replacement.

5.2. Update ownership comments in Flux kustomizations.
- Owner: `k8s-implementor`.
- Files: Flux kustomization files under `controllers`, `configs`, `apps`.
- Scope: clearly label template/shared, generated selected, user-global, and per-cluster ownership.

5.3. Add/adjust placeholder patch directories only where useful.
- Owner: `k8s-implementor`.
- Scope: use `.gitkeep` or README-style comments only if needed for empty dirs; avoid creating unused patch resources that break kustomize builds.

Validation gate after Phase 5:

- No references remain to deleted paths.
- Render check passes if Ansible render paths changed.
- Local Flux builds pass.

### Phase 6 — Documentation updates after implementation

Owners: `docs-writer` for `docs/` updates; top-level orchestrator or docs owner for `CLAUDE.md` if needed.

Tasks:

6.1. Update operational docs if the user-facing Flux customization workflow changed.
- Candidate files: `docs/adding-a-provider.md`, Flux/bootstrap docs if present, `README.md` if it documents Flux layout.
- Scope: explain where to put repo-wide Flux patches (`overlays/user`) and per-cluster patches (`overlays/clusters/kube1`).

6.2. Update `CLAUDE.md` only if implementation materially changes repo layout/status.
- Scope: update the Repository structure, Implementation status, and Flux specifics sections honestly.
- Do not claim live-cluster validation unless the user has actually run it.

6.3. ADR maintenance check.
- Owner: `adr-writer` only if needed.
- Scope: ADR-017 already records the decision. Update ADR-012 or write an ADR amendment only if implementation changes the documented decision rather than merely implementing it.

Validation gate after Phase 6:

- Docs mention the new ownership split without contradicting ADR-017.
- Docs retain generated-file and live-cluster safety warnings.

### Phase 7 — Review, fix loop, report

Owners: `k8s-reviewer`, `ansible-reviewer`, matching implementors for fixes, then `report-writer`.

Tasks:

7.1. Kubernetes/Flux review.
- Owner: `k8s-reviewer`.
- Scope: review `flux/**` changes for Flux/Kustomize correctness, path safety, pinned chart versions, adoption name preservation, `dependsOn`/`wait`/`prune`, and no accidental bootstrap-owned file edits.

7.2. Ansible render review, if Ansible changed.
- Owner: `ansible-reviewer`.
- Scope: review `infra/ansible/roles/config_render/**` changes for idempotency, FQCN, variable namespacing, check-mode drift behavior, generated headers, and no live/provider/secret operations.

7.3. Fix loop.
- Owner: matching implementor only, scoped to reviewer findings.
- Boundary: do not bundle unrelated cleanup into review fixes.

7.4. Closing report.
- Owner: `report-writer`.
- Scope: after implementation and reviews pass, write `.ai/reports/2026-06-25-flux-shared-overlay-ownership-migration.md` summarizing what changed, validation, and follow-ups.

## Validation required before handoff

Minimum non-live validation for Flux-only changes:

```bash
kubectl kustomize flux/infrastructure/controllers
kubectl kustomize flux/infrastructure/configs
kubectl kustomize flux/apps
```

If `kubectl kustomize` is unavailable, use `kustomize build` for the same paths.

Additional validation for Flux paths where feasible:

- Build direct overlay paths:
  - `flux/infrastructure/controllers/overlays/user`
  - `flux/infrastructure/controllers/overlays/clusters/kube1`
  - `flux/infrastructure/configs/overlays/clusters/kube1`
  - `flux/apps/overlays/clusters/kube1`
- Compare rendered controller output before/after migration with a local diff.
- Run `yamllint` on changed YAML if available.
- Run `kubeconform` on rendered manifests if available; tolerate/ignore Flux CRDs only if CRD schemas are unavailable and document the limitation.
- Run `flux build kustomization` only if the installed Flux CLI supports a local, non-cluster build for these paths; otherwise document why it was skipped.

Required validation when Ansible render changes are included:

```bash
cd infra/ansible
ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true
ansible-lint
```

Generated-output validation:

- Run an in-place render only when expected generated Flux files need updating:
  - `cd infra/ansible && ansible-playbook playbooks/render-config.yml -e config_cluster=kube1`
- Review `git diff` for generated Flux outputs.
- Confirm generated files retain `# Generated by render-config; do not edit.`.

Commands that must **not** be run by agents without explicit user approval:

- `flux reconcile ...`
- `kubectl apply`, `kubectl delete`, or live `kubectl get` against the cluster.
- `ansible-playbook ... bootstrap-flux.yml`.
- `ansible-playbook ... bootstrap-talos.yml` or provider playbooks unless explicitly requested.
- `hcloud`, `talosctl`, or commands that touch live infrastructure, provider APIs, secrets, or cluster state.

## Risks, side effects, cost-incurring operations, and live-cluster boundaries

- **Flux prune blast radius:** Changing `Kustomization.spec.path` can change the inventory scope used by Flux garbage collection. Prefer the top-level shim strategy first, or prove manifest equivalence before direct path switches.
- **Adoption contract risk:** Cilium and hcloud CCM are installed pre-Flux and adopted by Flux. Do not rename Helm releases, namespaces, charts, or alter values during layout migration.
- **Ordering risk:** Keep `infrastructure-controllers` before `infrastructure-configs` before `apps`. Preserve `dependsOn`, `wait`, and `prune` semantics unless a separate decision says otherwise.
- **Generated/source ownership confusion:** Do not leave render-owned selected resources in a path that future users reasonably interpret as human-authored `base/` unless comments are explicit or cleanup follows immediately.
- **Bootstrap-owned files:** `flux/clusters/<cluster>/flux-system/` is owned by `flux bootstrap`; avoid manual edits except deliberate bootstrap repair work outside this plan.
- **Docs drift:** ADR-010/011/012 and `CLAUDE.md` currently describe `_components/base/patches`; implementation may make those stale. Update docs only after the final layout is known.
- **No provider cost:** This migration should not create, delete, resize, or rebuild Hetzner resources. Any `hcloud` or provisioning command is out of scope and cost-incurring.
- **No live-cluster validation by agents:** Agents validate local renders/builds only. The user decides when to run live Flux or Ansible workflows.
- **Secrets:** Do not decrypt, print, or commit SOPS secrets, Flux private keys, hcloud tokens, Talos secrets, or generated machine configs.

## ADR needs

- ADR-017 already records the shared overlay ownership decision and the fact that Flux stays under `flux/`.
- ADR-012 may need a documentation update or amendment if implementation replaces the `catalog + render-owned base + single user overlay` layout with `components/generated-selected/overlays/user/overlays/clusters`.
- No new ADR is expected if implementation stays within ADR-017's accepted model.
- If a durable decision is made that contradicts ADR-017 or materially changes Flux bootstrap boundaries, pause and assign `adr-writer` to write/update the ADR after the decision is agreed.
