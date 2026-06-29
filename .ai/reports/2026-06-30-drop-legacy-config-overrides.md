# Report: Drop legacy `config/overrides/` tree — single ADR-016/017 source architecture

**Date:** 2026-06-30
**Plan:** `.ai/plans/2026-06-30-drop-legacy-config-overrides.md`
**Closes:** the "Legacy `config/overrides/` is still on disk" follow-up from `.ai/reports/2026-06-25-talos-config-boundary-migration.md` (§Remaining) and `.ai/reports/2026-06-29-talos-deviceselector-schema-narrowing.md` (§Out of scope / §Follow-ups)
**Type:** Refactor + config cleanup — no live cluster operations, no behavior change on rendered output, no provider-costing actions

---

## What changed

The dormant legacy `config/overrides/` fallback tree and all deprecated input paths are removed. The bootstrap template now operates **exclusively** on the ADR-016/017 architecture: `config/defaults/cluster.yaml` (template-owned) deep-merged with `config/clusters/<cluster>/{cluster,nodes}.yaml` (user-owned, selected via `-e config_cluster=<name>`, default `kube1`). The Talos-shaped per-node overlay path `config/clusters/<cluster>/talos/nodes/*.yaml` remains the only place per-node Talos configuration lives.

Three legacy mechanisms were removed in one slice, all AGREED in the plan:

1. **`config/overrides/` tree + loader fallback** — the core cleanup. The loader no longer stat-guesses between two trees; it directly reads the selected-cluster files. A missing `config/clusters/<cluster>/cluster.yaml` fails fast with a clear error naming the path and the `-e config_cluster=<name>` selector. A missing `nodes.yaml` resolves to `{nodes: []}` (no fallback, not an error).
2. **`nodes[].patch` raw-fragment per-node render path** — deprecated by ADR-016, fully removed. The render block, the `config_render_nodes_with_patches` fact, and the `"patch": {}` schema property are all gone. Per-node Talos config lives ONLY in `config/clusters/<cluster>/talos/nodes/*.yaml`; `nodes.yaml` keeps node identity facts (`name`, `role`, `network`) only.
3. **Deprecated-key rejection guards** in `validate.yml` — **KEPT** as permanent template-UX guards (`features.cni`, `features.hcloudCcm`, `talos.install`, `talos.patches`, `talos.network.vip`). Only the comment headers were reframed from "migration/transitional" to permanent "unsupported key" language. No assert logic, no fail_msg target paths changed.

The graceful `stat`-guards in `render_talos.yml` that tolerate a cluster lacking some Talos overlay files were preserved (good template behavior for a fresh cluster) — only their misleading "legacy config/overrides/ path" comments were reworded.

---

## Files changed (commitable)

### Render role / playbooks (5 files) — Task A
- `infra/ansible/roles/config_render/defaults/main.yml` — deleted `config_render_overrides_path` and `config_render_nodes_path`; moved `config_render_defaults_path` out of the "Legacy overrides fallback paths" section and relabeled it as a primary input.
- `infra/ansible/roles/config_render/tasks/load.yml` — replaced L23–99 (stat → fallback → resolve machinery) with direct selected-cluster reads + a fail-fast `assert` naming `config/clusters/<cluster>/cluster.yaml` and the `-e config_cluster=<name>` selector. `check_mode: false` preserved on `stat`/`slurp`/`assert` so `--check` render still loads inputs. `config_render_overrides` / `config_render_nodes` fact names preserved to keep `validate.yml` references intact (R5 honored).
- `infra/ansible/roles/config_render/tasks/render_talos.yml` — deleted the legacy `nodes[].patch` block (L643–687) and the `config_render_nodes_with_patches` fact; the subsequent stale-file cleanup was adjusted to no longer reference the removed fact; four "legacy config/overrides/ path" comments reworded.
- `infra/ansible/roles/config_render/tasks/render_ansible.yml` — L15 comment repointed at `config/clusters/<cluster>/cluster.yaml`.
- `infra/ansible/playbooks/render-config.yml` — L6/L25–26 header comments repointed at `config/clusters/<cluster>/`.

### Config + schema (5 files / entries) — Task B
- **Deleted:** `config/overrides/cluster.yaml`, `config/overrides/nodes.yaml`, and the now-empty `config/overrides/` directory.
- `config/schemas/nodes.schema.json` — removed the `"patch": {}` property; kept `name`, `role`, `network`; `additionalProperties: true` retained so identity facts pass.
- `config/clusters/kube1/nodes.yaml` — dropped the `patch:` example line from the comment; the comment now points at `config/clusters/kube1/talos/nodes/*.yaml` for per-node Talos config.
- `config/clusters/kube1/cluster.yaml` — L9–12 header rewritten; the legacy-fallback paragraph is gone, replaced with a statement that `config/clusters/kube1/` is the sole user-owned source for this cluster.

### validate.yml comment reframe (1 file) — Task C
- `infra/ansible/roles/config_render/tasks/validate.yml` — four comment blocks (L132–135, L160–164, L202–207, L219–226) reworded from "stale/transitional key" / "migration" headers to permanent "unsupported key" guards. **No assert logic, no fail_msg target paths changed.**

### Docs + ADRs — Task D
- `docs/node-roles.md` — dropped the L9–11 legacy `config/overrides/` note.
- `docs/upgrades.md` — dropped the L20–21 legacy reference.
- `docs/adding-a-provider.md` — three `config/overrides/` refs (L75, L82, L124) repointed at `config/clusters/<cluster>/`.
- `AGENTS.md` — verified, no `config/overrides` references (no changes needed).
- ADR addenda appended:
  - **ADR-007** — the "values stay in `config/overrides/`" line at L23 is now historical; addendum notes the path no longer exists.
  - **ADR-011** — the `defaults/`+`overrides/` two-layer is now `defaults/` + `clusters/<cluster>/`.
  - **ADR-015** — L21 reference updated.
  - **ADR-016** — implementation addendum: legacy `config/overrides/` removed + `nodes[].patch` retired on 2026-06-30, with the `config_render_overrides` fact name kept (sourced from the cluster file) as a deliberate R5 risk-mitigation against a wider edit.

---

## What was validated

All local, read-only, no live cluster contact, no provider-costing commands, no SOPS decryption. Four parallel implementor tasks (A/B/C/D) touched disjoint files; ansible-reviewer was run on Tasks A–C.

| Gate | Result |
|---|---|
| `ansible-lint` (production profile) | 0 failures, 0 warnings |
| `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true` (run 1) | "No render drift detected. Committed generated files match render output." |
| Same render, run 2 (determinism) | "No render drift detected" — clean re-run |
| `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check` | PASS |
| `ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check` | PASS |
| Negative test: `ansible-playbook playbooks/render-config.yml -e config_cluster=bogus` | **Fails fast with the new clear error** — no silent fallback, no `config/overrides/*` read attempted. Error message names `config/clusters/bogus/cluster.yaml` and the `-e config_cluster=<name>` selector. |
| `graphify update .` | knowledge graph refreshed |
| Stale cleanup in `render_talos.yml` (R4) | `config_render_nodes_with_patches` reference removed in the same task; generated `nodes/` directory cleanup remains correct |
| `check_mode: false` on new `stat`/`slurp`/`assert` tasks (R3) | Preserved — `--check` render still loads inputs |
| `config_render_overrides` fact name reused for cluster-file source (R5) | Honored — `validate.yml` references stay valid; no wider edit |
| Deprecated-key guards in `validate.yml` (`features.cni` / `features.hcloudCcm` / `talos.install` / `talos.patches` / `talos.network.vip`) | Asserts unchanged; only comment headers reworded |

### Reviewer findings and fixes

The ansible-reviewer found issues in Tasks A–C, all fixed by the implementors before handoff (no report to user until clean):

| # | Severity | Finding | Fix |
|---|---|---|---|
| 1 | HIGH | New fail-fast `assert` in `load.yml` did not preserve `check_mode: false`, breaking `--check` render. | `check_mode: false` set on the new `stat` / `slurp` / `assert` tasks (R3). |
| 2 | MEDIUM | Stale cleanup task in `render_talos.yml` still referenced the removed `config_render_nodes_with_patches` fact. | Reference removed; generated `nodes/` cleanup verified clean (R4). |
| 3 | LOW | A "legacy config/overrides/ path" comment survived in `render_talos.yml` outside the four flagged locations. | Comment reworded in the same task. |

---

## What was NOT run and why

- **No `bootstrap-talos.yml` without `--syntax-check`, no `bootstrap-flux.yml`, no `provision-infra.yml`, no `image-talos.yml`, no `upgrade-talos.yml`.** Live cluster apply. Per the plan's safety boundary, the user runs anything live in their own terminal; agents do not.
- **No `hcloud` API call, no `talosctl apply-config` / `talosctl upgrade` / `talosctl patch machineconfig`, no `kubectl` against a live cluster, no `flux reconcile`, no local `talosctl gen config` validation against rendered outputs.** Same reason.
- **No follow-up Talos deviceSelector live-apply** — that was the prior slice's gated step (the `86:00:00:*` glob) and remains the highest-risk pending item. This slice touches no Talos node network config; the deviceSelector and VIP binding are unchanged.

---

## Remaining / follow-ups

- **Live cluster state is unverified.** As with prior slices, "Working" in AGENTS.md / CLAUDE.md means the code is built, reviewed, and passes validation — not that any live apply has been observed. The same live-validation commands from the prior slice's reports apply (Cilium `daemonset/cilium` Ready, no `uninitialized` taint, every `providerID` starts with `hcloud://`, `flux-system` pods Ready, all HelmReleases Ready without install/uninstall churn). **For this slice specifically, no live apply is gated** — `config_cluster: kube1` is the default, the active source has not changed, and the render drift gate has already proven equivalence.
- **The pre-existing `86:00:00:*` deviceSelector live-apply remains the highest-risk pending item** (see `.ai/reports/2026-06-29-talos-deviceselector-schema-narrowing.md` §Follow-ups). Unchanged by this slice.
- **Schema surface is not yet fully narrowed.** `features.cni` / `features.hcloudCcm` / `talos.network.vip` / `talos.network.defaultInterface` / `talos.version` / `talos.extensions` / `storage.volumes` remain in `config/schemas/cluster-input.schema.json` and are still wired through. The transitional keys rejected by `validate.yml` are now the only ones that produce hard errors for legacy values. A full schema-narrowing pass is a natural follow-up if/when more keys are retired.
- **ADR-017's `overlays/user/` layer is explicitly out of scope** (plan §Premises #6). The model stays two-layer (defaults + per-cluster) for this slice. If a future user-global overlay tier is needed, it is a separate ADR.
- **Reviewer note — out-of-scope finding from the prior slice still open:** the untracked `infra/ansible/infra/talos/secrets/talosconfig` file flagged in `.ai/reports/2026-06-29-talos-deviceselector-schema-narrowing.md` §Follow-ups. Not touched by this slice — still flagged for user decision.

---

## Artifacts

- Plan: `.ai/plans/2026-06-30-drop-legacy-config-overrides.md`
- Predecessor reports: `.ai/reports/2026-06-25-talos-config-boundary-migration.md`, `.ai/reports/2026-06-29-talos-deviceselector-schema-narrowing.md`
- ADR addenda: `docs/cluster-bootstrap/adrs/007-provider-restructuring.md`, `011-*.md`, `015-*.md`, `016-talos-config-boundary.md` (implementation addendum 2026-06-30)
- Modified (Ansible): `infra/ansible/roles/config_render/{defaults/main,tasks/load,tasks/validate,tasks/render_talos,tasks/render_ansible}.yml`, `infra/ansible/playbooks/render-config.yml`
- Modified (config): `config/schemas/nodes.schema.json`, `config/clusters/kube1/{cluster,nodes}.yaml`
- Modified (docs): `docs/node-roles.md`, `docs/upgrades.md`, `docs/adding-a-provider.md`
- **Deleted:** `config/overrides/cluster.yaml`, `config/overrides/nodes.yaml`, the `config/overrides/` directory itself, the `nodes[].patch` render block in `render_talos.yml`, and the `config_render_nodes_with_patches` fact
