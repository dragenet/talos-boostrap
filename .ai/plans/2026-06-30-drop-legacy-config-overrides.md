# Plan: Drop legacy `config/overrides/` tree — single ADR-016/017 source architecture

Date: 2026-06-30
Status: Ready to execute (design AGREED — do not re-litigate)
Closes: the "Legacy `config/overrides/` is still on disk" follow-up from
`.ai/reports/2026-06-25-talos-config-boundary-migration.md` (§Remaining) and
`.ai/reports/2026-06-29-talos-deviceselector-schema-narrowing.md` (§Out of scope)

---

## Goal

Remove the dormant legacy `config/overrides/` fallback tree and all deprecated
input paths so the bootstrap template operates **exclusively** on the ADR-016/017
architecture: `config/defaults/cluster.yaml` (template-owned) deep-merged with
`config/clusters/<cluster>/{cluster,nodes}.yaml` (user-owned, selected via
`-e config_cluster=<name>`, default `kube1`).

**Expected render-output drift: NONE.** `config_cluster: kube1` is already the
default and `config/clusters/kube1/*` is already the active source; the
`config/overrides/` path is only read when the selected-cluster files are absent,
which never happens. The Phase 6 check-mode render is the drift guard that proves
this.

### Three legacy mechanisms removed (all AGREED)

1. **`config/overrides/` tree + loader fallback** — the core cleanup.
2. **`nodes[].patch` raw-fragment per-node render path** — deprecated by ADR-016 in
   favour of Talos-shaped overlays under `config/clusters/<cluster>/talos/nodes/`.
   **Removed fully** (render block + schema key).
3. **Deprecated-key rejection guards** in `validate.yml` — **KEPT** as permanent
   template-UX guards; only comments reframed from "migration" to "unsupported".

---

## Premises / AGREED DECISIONS (bake in; do not reopen)

1. **Remove `config/overrides/` entirely** — both `cluster.yaml` and `nodes.yaml`.
   No deprecation window; the path is already dormant.
2. **Loader fails fast** when the selected-cluster `cluster.yaml` is missing (clear
   error naming `config/clusters/<cluster>/cluster.yaml`). A missing
   `nodes.yaml` resolves to `{nodes: []}` (no fallback, not an error).
3. **`nodes[].patch` removed fully** — delete the legacy render block + the
   `"patch": {}` schema property. Per-node Talos config lives ONLY in
   `config/clusters/<cluster>/talos/nodes/*.yaml`. `nodes.yaml` keeps node identity
   facts (`name`, `role`, `network`).
4. **KEEP the deprecated-key guards** in `validate.yml` (`features.cni`,
   `features.hcloudCcm`, `talos.install`, `talos.patches`, `talos.network.vip`).
   Reword comments only — no functional change.
5. **Full docs/ADR treatment** — fix all `config/overrides` references, add ADR
   addenda, and write a closing report (repo docs-first standard).
6. **No user-global overlay tier added.** ADR-017's `overlays/user/` layer is out of
   scope; the model stays two-layer (defaults + per-cluster) for this slice.

### KEEP (explicitly out of scope — do not touch)
- `config/defaults/cluster.yaml` as the always-loaded template defaults layer.
- The graceful `stat`-guards in `render_talos.yml` that tolerate a cluster lacking
  some Talos overlay files (good template behaviour for a fresh cluster) — only
  their misleading "legacy config/overrides/ path" comments change.
- The Talos-shaped per-node overlay path (`config/clusters/<cluster>/talos/nodes/`)
  and its render block — this is the replacement, not legacy.
- All generated output trees (`infra/ansible/.../group_vars/`,
  `infra/talos/patches/generated/`, `flux/.../generated/selected/`).

---

## Current-state facts (gathered read-only — do NOT re-investigate)

- `config_render` role defaults (`defaults/main.yml`):
  - L28 `config_render_defaults_path` → `config/defaults/cluster.yaml` (KEEP, relabel).
  - L29 `config_render_overrides_path` → `config/overrides/cluster.yaml` (DELETE).
  - L30 `config_render_nodes_path` → `config/overrides/nodes.yaml` (DELETE).
  - Selected-cluster paths (L20–22, L34–39) already present and correct.
- `tasks/load.yml` (119 lines): L23–99 is the stat → fallback → resolve machinery.
  Replace with a direct read of the selected-cluster files; keep the parse +
  deep-merge tail (L101–119). `config_render_overrides` fact stays (now sourced
  only from the cluster file) so `validate.yml` need not change its references.
- `tasks/render_talos.yml`: legacy comments at L139/L299/L489/L578; legacy
  `nodes[].patch` block at **L643–687** (`config_render_nodes_with_patches` fact +
  "Render legacy per-node patch files"). The Talos-shaped per-node path (L577–641)
  stays. Verify the stale-file cleanup after L688 does not solely depend on the
  removed fact before deleting.
- `tasks/render_ansible.yml` L15: comment references `config/overrides/cluster.yaml`.
- `playbooks/render-config.yml` L6, L25–26: header references the legacy fallback.
- `config/schemas/nodes.schema.json` L18: `"patch": {}` (DELETE). Keep `name`,
  `role`, `network`; `additionalProperties: true` stays so identity facts pass.
- `config/clusters/kube1/nodes.yaml`: `nodes: []`; comment block shows a `patch:`
  example to update.
- `config/clusters/kube1/cluster.yaml` L9–12: header documents the legacy fallback.
- `validate.yml`: deprecated-key guards at L132–158 (features.cni / hcloudCcm),
  L209–217 (talos.network.vip), L219–270 (talos.install / talos.patches). KEEP;
  reword comments only. The `config_render_overrides.*` references stay valid.
- Canonical status doc is **`AGENTS.md`** at repo root (there is no `CLAUDE.md`).
- No CI/pre-commit enforces render — Phase 6 checks are run locally by the user/agent.

---

## Work breakdown (implementor tasks — split by non-overlapping files)

### Task A — Render role: loader + Talos render (ansible-implementor)
Files: `infra/ansible/roles/config_render/defaults/main.yml`,
`infra/ansible/roles/config_render/tasks/load.yml`,
`infra/ansible/roles/config_render/tasks/render_talos.yml`,
`infra/ansible/roles/config_render/tasks/render_ansible.yml`,
`infra/ansible/playbooks/render-config.yml`.

1. `defaults/main.yml`: delete `config_render_overrides_path` +
   `config_render_nodes_path`; move `config_render_defaults_path` out of the
   "Legacy overrides fallback paths" section and relabel it as a primary input.
2. `load.yml`: remove L23–99; replace with:
   - read defaults (unchanged);
   - `assert` the selected-cluster `cluster.yaml` exists (fail_msg names the path
     and the `-e config_cluster=<name>` selector);
   - `slurp` `cluster.yaml`;
   - `stat` + conditional `slurp` of `nodes.yaml`; when absent set
     `config_render_nodes_raw` to an inline `{nodes: []}` equivalent (no legacy read);
   - keep the parse + deep-merge + nodes-list tail. Preserve the
     `config_render_overrides` / `config_render_nodes` fact names.
3. `render_talos.yml`: delete the legacy `nodes[].patch` block (L643–687) and the
   `config_render_nodes_with_patches` fact; confirm/adjust the subsequent
   stale-file cleanup so it no longer references the removed fact; reword the four
   "legacy config/overrides/ path" comments (keep the stat-guards).
4. `render_ansible.yml` L15 + `render-config.yml` L6/L25–26: repoint comments at
   `config/clusters/<cluster>/cluster.yaml`; drop fallback wording.

### Task B — Config + schema (ansible-implementor; no file overlap with A)
Files: delete `config/overrides/cluster.yaml`, `config/overrides/nodes.yaml`;
edit `config/schemas/nodes.schema.json`, `config/clusters/kube1/nodes.yaml`,
`config/clusters/kube1/cluster.yaml`.

1. Delete the two `config/overrides/*` files (and the now-empty dir).
2. `nodes.schema.json`: remove the `"patch": {}` property.
3. `config/clusters/kube1/nodes.yaml`: drop the `patch:` example line; point the
   comment at `config/clusters/kube1/talos/nodes/*.yaml` for per-node Talos config.
4. `config/clusters/kube1/cluster.yaml`: rewrite the L9–12 header note (remove the
   legacy-fallback paragraph; state `config/clusters/kube1` is the sole source).

### Task C — validate.yml comment reframe (ansible-implementor; isolated file)
File: `infra/ansible/roles/config_render/tasks/validate.yml`.
- Reword the "stale/transitional key" / "migration" comment headers (L132–135,
  L160–164, L202–207, L219–226) to read as permanent "unsupported key" guards.
  **No assert logic, no fail_msg target paths changed.**

### Task D — Docs + ADRs (docs-writer + adr-writer; isolated from code tasks)
- `docs/node-roles.md` (drop L9–11 legacy note), `docs/upgrades.md` (drop L20–21),
  `docs/adding-a-provider.md` (L75, L82, L124 → `config/clusters/<cluster>/`),
  `AGENTS.md` (any `config/overrides` refs + repo-structure tree),
  `config/clusters/kube1/cluster.yaml` header already handled in Task B.
- ADR addenda (adr-writer): **ADR-011** (the `defaults/`+`overrides/` two-layer is
  now `defaults/` + `clusters/<cluster>/`), **ADR-007** (L23
  "values stay in config/overrides/"), **ADR-015** (L21), **ADR-016**
  (implementation addendum: legacy `config/overrides/` removed + `nodes[].patch`
  retired on 2026-06-30).

> Task D is documentation-only and may run in parallel with A/B/C. Tasks A, B, C
> touch disjoint files and may run in parallel; none edits the same file.

---

## Phase 6 — Validation (local / read-only; the USER runs anything live)

Run from `infra/ansible/` unless noted. Agents must NOT contact the live cluster,
hcloud API, or git remote.

1. `ansible-lint` (production profile) — expect 0 failures / 0 warnings.
2. `ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true`
   — **MUST report "No render drift detected."** (the core safety proof). Run twice;
   second run must also be clean (determinism).
3. `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check` — PASS.
4. `ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check` — PASS.
5. **Negative test:** `ansible-playbook playbooks/render-config.yml -e config_cluster=bogus`
   — MUST fail fast with the new clear error (no silent fallback, no overrides read).
6. `graphify update .` to refresh the knowledge graph.

## Review + close
- `ansible-reviewer` on Tasks A–C (idempotency, FQCN, no behaviour drift, fail-fast
  correctness, no live-cluster contact).
- `report-writer` closing report under `.ai/reports/2026-06-30-drop-legacy-config-overrides.md`.
- Ask the user before any `git` commit.

---

## Risks / watch-outs

- **R1 — Hidden render drift.** Low: the new path is already active. Phase 6 step 2
  is the gate; if drift appears, STOP and diff before proceeding — it means
  `config/overrides/*` was silently diverging from `config/clusters/kube1/*`.
- **R2 — `nodes[].patch` consumers.** None today (`nodes: []`). Removing the render
  block + schema key is safe; documented as retired in the ADR-016 addendum.
- **R3 — Loader fail-fast regressions.** Ensure `check_mode: false` is preserved on
  the new `stat`/`slurp`/`assert` tasks so `--check` render still loads inputs.
- **R4 — Stale-cleanup task in render_talos.yml** may reference the removed
  `config_render_nodes_with_patches` fact — verify and fix in the same task to keep
  the generated `nodes/` directory cleanup correct.
- **R5 — `config_render_overrides` fact name** is reused by `validate.yml`. Keep the
  name when sourcing from the cluster file to avoid a wider edit; do not rename in
  this slice.
