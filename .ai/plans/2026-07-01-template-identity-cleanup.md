# Template Identity Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `kube1-orchestration` for routing and `kube1-validation` before handoff. Use focused implementor/reviewer agents where edits cross Ansible/Talos and Flux/Kubernetes domains.

**Goal:** Remove active `kube1` identity from the repository so the current tree reads as a reusable Talos/Kubernetes deployment template, while preserving historical artifacts and keeping `docs/` as the documentation root.

**Architecture:** Perform a current-tree cleanup only. Parameterize or replace active `kube1` defaults, paths, docs, schemas, and examples with template-neutral names while leaving historical `.ai/` reports/plans intact unless an untracked/current plan is explicitly in scope. Do not rewrite git history in this implementation plan.

**Tech Stack:** Talos Linux, Kubernetes, Flux, Kustomize, Helm, Ansible, JSON Schema, Markdown docs.

## Global Constraints

- Keep `docs/` as the documentation root.
- Do not touch `dev-docs/`; its future role is not clarified.
- Do not rewrite `.ai/` historical reports or plans as part of the main cleanup.
- Do not run live cluster, provider-costing, SOPS decryption, `talosctl apply-*`, `kubectl` against a live cluster, `flux reconcile`, or history-rewriting commands.
- Do not run `git filter-repo`, `git push --force`, or any destructive git operation. History rewriting is a separate explicit approval path.
- Preserve user/unrelated work. At plan creation time, `.ai/plans/2026-06-30-app-entrypoints.md` was already untracked and must not be overwritten or deleted.
- Treat `kube1` in archived graphify outputs or historical `.ai` artifacts as historical unless the user separately asks for archive cleanup.

---

## File Map

- `README.md`: top-level repository identity and layout description.
- `AGENTS.md`: OpenCode-facing repo guidance currently tied to `kube1`; update only if it describes the active repo identity rather than historical workflow names.
- `.opencode/skills/kube1-*`: active local agent skills. Either keep names for now and update copy minimally, or defer renaming skills to a separate OpenCode configuration task because skill renames affect agent routing.
- `opencode.jsonc`: local permissions; remove stale path references to `~/Projects/infra/kube1/**` if they are active and safe to neutralize.
- `config/schemas/*.json`: schema `$id`, titles, descriptions, and examples should be template-neutral.
- `config/defaults/cluster.yaml`: default cluster name and comments should be template-neutral.
- `config/clusters/kube1/**`: active cluster example inputs. Decide whether this remains a concrete example cluster directory renamed to `example`, or keep directory path and only remove textual `kube1` values. The recommended implementation is to rename to `config/clusters/example/**` if renderer defaults are updated in the same task.
- `flux/clusters/kube1/**`: active Flux cluster entrypoint. Rename to `flux/clusters/example/**` only if Flux references, docs, and render outputs are updated together.
- `flux/apps/README.md` and `flux/clusters/*/README.md`: remove hardcoded `kube1` examples from active guidance.
- `docs/**/*.md`: replace active user-facing `kube1` identity with template-neutral wording. Keep historical rationale inside kube1-specific ADRs unless the file itself is being converted to a template-neutral example.
- `.ai/plans/2026-06-30-app-entrypoints.md`: untracked current artifact containing `kube1`; review separately at the end and ask user whether to update, keep, or delete.

## Scope Decisions

### Current-Tree Cleanup

The main cleanup should update tracked active files only. It should not attempt to make old reports, old plans, graphify snapshots, or commit history appear as though `kube1` never existed.

### Concrete Example Name

Use `example` as the neutral concrete cluster name when a real directory, rendered path, or default value must exist. Use `<cluster>` in prose where describing user-provided names.

### ADR Handling

No ADR is required for a mechanical identity cleanup. If implementation changes the durable contract for cluster directory names, Flux cluster entrypoints, or template/user ownership boundaries, update an existing provider-agnostic ADR under `docs/cluster-bootstrap/adrs/` with an implementation addendum before handoff.

## Task 1: Inventory Active `kube1` References

**Files:**
- Read/search only in this task.

**Interfaces:**
- Produces: a categorized reference list for later tasks: active config, active Flux, docs, local tooling, historical archive.

- [ ] **Step 1: Capture current status**

Run:

```bash
git status --short
```

Expected: note existing unrelated/untracked files. Do not modify them.

- [ ] **Step 2: Search active non-historical files for `kube1`**

Run:

```bash
rg -n "\bkube1\b|kube1[-_/]" README.md AGENTS.md opencode.jsonc config flux docs .opencode --glob '!graphify-out/**'
```

Expected: references grouped by active domain. Do not include `.ai/` in the main count.

- [ ] **Step 3: Search current `.ai` separately**

Run:

```bash
rg -n "\bkube1\b|kube1[-_/]" .ai --glob '*.md'
```

Expected: many historical references plus the untracked app-entrypoints plan. Record them as historical/deferred unless the user asks otherwise.

- [ ] **Step 4: Search old `docs/` path references**

Run:

```bash
rg -n "\bdev-docs/|\bdocs/" README.md AGENTS.md config flux docs .opencode .ai --glob '*.md'
```

Expected: `docs/` remains valid. `dev-docs/` should not be introduced or edited.

## Task 2: Neutralize Root Repository Identity

**Files:**
- Modify: `README.md`
- Modify if active identity requires it: `AGENTS.md`
- Modify if stale local permission exists: `opencode.jsonc`

**Interfaces:**
- Consumes: Task 1 root reference list.
- Produces: template-neutral top-level identity.

- [ ] **Step 1: Update README title and summary**

Change `README.md` from a `kube1` cluster README to a template README. Recommended wording:

```markdown
# Talos Kubernetes Bootstrap Template

Infrastructure-as-code template for deploying Talos Linux Kubernetes clusters.
The current implementation includes Ansible provisioning/bootstrap flows, Talos
configuration rendering, and Flux GitOps manifests.
```

- [ ] **Step 2: Keep docs root unchanged**

Ensure the repository layout still says:

```markdown
- `docs/` — architecture decisions and runbooks
```

Do not change it to `dev-docs/`.

- [ ] **Step 3: Update AGENTS only for active repo identity**

If `AGENTS.md` says the repo is for `kube1`, change that line to template-neutral wording. Keep workflow/subagent guidance intact unless it directly claims the active product is `kube1`.

- [ ] **Step 4: Remove stale local permission path if safe**

If `opencode.jsonc` contains an allow path like `~/Projects/infra/kube1/**`, replace it with the current repo path or remove it only if it is clearly stale and unrelated to active commands. Do not broaden permissions.

- [ ] **Step 5: Verify root references**

Run:

```bash
rg -n "\bkube1\b|kube1[-_/]" README.md AGENTS.md opencode.jsonc
```

Expected: no active repository identity claim remains. Any remaining match must be an intentional skill/agent name or documented historical note.

## Task 3: Neutralize Config Cluster Identity

**Files:**
- Modify: `config/defaults/cluster.yaml`
- Modify: `config/schemas/cluster-input.schema.json`
- Modify: `config/schemas/nodes.schema.json`
- Rename or modify: `config/clusters/kube1/**`
- Modify Ansible defaults/templates only if they hardcode `kube1` as default cluster name.

**Interfaces:**
- Consumes: Task 1 config reference list.
- Produces: renderable template config with neutral example cluster identity.

- [ ] **Step 1: Decide concrete path strategy from local references**

Prefer renaming `config/clusters/kube1/` to `config/clusters/example/` if renderer defaults and docs can be updated in the same task. If too many generated or Flux dependencies assume `kube1`, leave the directory path for a follow-up and remove textual `kube1` values only. Record the decision in the task notes.

- [ ] **Step 2: Update schema metadata**

Change schema `$id` values from `https://kube1.local/...` to a neutral local template URI such as:

```json
"$id": "https://talos-bootstrap-template.local/schemas/cluster-input.schema.json"
```

Use matching neutral title values, for example:

```json
"title": "Cluster input"
```

and:

```json
"title": "Per-node overrides"
```

- [ ] **Step 3: Update default cluster name**

Search for `config_cluster: kube1`, `cluster.name: kube1`, and examples using `kube1`. Replace defaults that select a concrete cluster with `example` only if `config/clusters/example/` exists after Step 1.

- [ ] **Step 4: Update config comments and examples**

Replace comments like `config/clusters/kube1/...` with `config/clusters/<cluster>/...` in prose. Replace sample hostnames like `kube1-hb1` with `example-hb1` or `<cluster>-hb1` depending on whether it is YAML example data or explanatory prose.

- [ ] **Step 5: Render-check config locally**

If the default concrete cluster becomes `example`, run:

```bash
cd infra/ansible && ansible-playbook playbooks/render-config.yml -e config_cluster=example -e config_render_check=true
```

Expected: either pass with no render drift or fail with actionable references to old `kube1` paths to fix in this task.

If the directory remains `kube1` temporarily, run the existing check:

```bash
cd infra/ansible && ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true
```

Expected: pass with no render drift.

## Task 4: Neutralize Flux Cluster Identity

**Files:**
- Rename or modify: `flux/clusters/kube1/**`
- Modify: `flux/apps/README.md`
- Modify: any Flux README or Kustomization path that hardcodes `kube1`
- Modify generated/render templates only if they produce `flux/clusters/kube1` by default.

**Interfaces:**
- Consumes: Task 1 Flux reference list and Task 3 concrete cluster-name decision.
- Produces: Flux tree that builds using the same neutral concrete cluster name as config.

- [ ] **Step 1: Align Flux cluster directory with config strategy**

If Task 3 renamed to `example`, rename `flux/clusters/kube1/` to `flux/clusters/example/` and update all references. If Task 3 kept `kube1`, keep this path and only neutralize README prose/examples.

- [ ] **Step 2: Update per-app convention docs**

In `flux/apps/README.md`, replace hardcoded examples like:

```markdown
flux/clusters/kube1/<app>.yaml
overlays/clusters/kube1/
```

with:

```markdown
flux/clusters/<cluster>/<app>.yaml
overlays/clusters/<cluster>/
```

- [ ] **Step 3: Update cluster README links**

In the Flux cluster README, keep ADR links pointing to `docs/` and remove `kube1` as the named active cluster unless the file remains an example cluster directory.

- [ ] **Step 4: Build Flux locally**

Run the appropriate command based on the directory decision:

```bash
kustomize build flux/clusters/example
```

or:

```bash
kustomize build flux/clusters/kube1
```

Expected: build succeeds or fails only on pre-existing empty/no-resource behavior already documented in the repo. Fix path errors introduced by this task.

## Task 5: Neutralize Active Documentation

**Files:**
- Modify: `docs/**/*.md`
- Do not modify: `dev-docs/**`
- Do not modify historical `.ai/**` in this task.

**Interfaces:**
- Consumes: Task 1 docs reference list and Task 3/4 naming decision.
- Produces: docs that describe a template and use `docs/` links.

- [ ] **Step 1: Classify docs references**

Classify each `docs/**/*.md` `kube1` match as one of:

```text
active template instruction
example value
historical kube1-specific ADR
```

Only rewrite active template instructions and example values. Do not rewrite a kube1-specific ADR into false history unless the file is intentionally being converted to a template ADR.

- [ ] **Step 2: Update active runbooks and onboarding docs**

Use `<cluster>` in prose and `example` for concrete examples. For example:

```markdown
config/clusters/<cluster>/cluster.yaml
```

and:

```yaml
example-hb1:
  node_role: hybrid
```

- [ ] **Step 3: Preserve docs root links**

Keep links such as:

```markdown
docs/cluster-bootstrap/adrs/014-node-discovery-dynamic-inventory.md
```

Do not change them to `dev-docs/`.

- [ ] **Step 4: Check markdown references**

Run:

```bash
rg -n "\bdev-docs/" docs README.md AGENTS.md flux config
```

Expected: no matches unless pre-existing and intentionally documented as out of scope.

## Task 6: Review Local Agent/Skill Naming Separately

**Files:**
- Read/possibly modify: `.opencode/skills/kube1-*`
- Read/possibly modify: `AGENTS.md`

**Interfaces:**
- Consumes: active agent references from Task 1.
- Produces: a recommendation, not necessarily edits.

- [ ] **Step 1: Do not rename skills in the same pass**

Skill names like `kube1-orchestration` are part of local OpenCode routing. Renaming them is an OpenCode configuration change and should use the `customize-opencode` skill in a separate task if needed.

- [ ] **Step 2: Update visible copy only if safe**

If a skill file contains prose like "this repo is infrastructure for kube1", change the prose to template-neutral wording only if that does not require renaming the skill or changing agent dispatch names.

- [ ] **Step 3: Record follow-up if names remain**

If `rg` still finds `kube1` only in skill names or agent names, record that as an intentional follow-up: "rename kube1 local OpenCode skills after template cleanup".

## Task 7: Validation Sweep

**Files:**
- No planned edits unless validation finds missed active references.

**Interfaces:**
- Consumes: outputs of Tasks 2-6.
- Produces: verification evidence for handoff.

- [ ] **Step 1: Search active tree**

Run:

```bash
rg -n "\bkube1\b|kube1[-_/]" README.md AGENTS.md opencode.jsonc config flux docs .opencode --glob '!graphify-out/**'
```

Expected: remaining matches are only explicitly accepted follow-ups or historical ADR content.

- [ ] **Step 2: Search accidental `dev-docs` references**

Run:

```bash
rg -n "\bdev-docs/" README.md AGENTS.md config flux docs .opencode .ai --glob '*.md'
```

Expected: no new active references.

- [ ] **Step 3: Run relevant local checks**

Run local checks appropriate to touched files:

```bash
cd infra/ansible && ansible-lint
```

Run render check using the concrete cluster selected in Task 3:

```bash
cd infra/ansible && ansible-playbook playbooks/render-config.yml -e config_cluster=example -e config_render_check=true
```

or, if the directory remains `kube1` temporarily:

```bash
cd infra/ansible && ansible-playbook playbooks/render-config.yml -e config_cluster=kube1 -e config_render_check=true
```

Run Flux build using the concrete cluster selected in Task 4:

```bash
kustomize build flux/clusters/example
```

or:

```bash
kustomize build flux/clusters/kube1
```

Expected: commands pass, or any failures are documented with exact error output and no success claim is made.

## Task 8: Report and Handoff

**Files:**
- Create: `.ai/reports/2026-07-01-template-identity-cleanup.md`

**Interfaces:**
- Consumes: final diff and validation evidence.
- Produces: concise closeout report and explicit follow-ups.

- [ ] **Step 1: Write report**

Report must include:

```markdown
# Template identity cleanup closing report

## What changed

## What was validated

## What was explicitly not run

## Remaining kube1 references

## Follow-ups / risks
```

- [ ] **Step 2: Call out intentionally untouched areas**

The report must explicitly say:

```markdown
- Historical `.ai/` plans/reports were not rewritten.
- `dev-docs/` was not touched.
- Git history was not rewritten.
```

- [ ] **Step 3: Ask before commit**

After implementation, review, validation, and report, ask the user whether to commit. Do not commit without explicit approval.

## History Rewrite Option: Deferred

If the user later decides that historical `kube1` references must be removed from commits, use a separate plan. Read the current official `git-filter-repo` documentation again before executing. Required safety boundaries for that separate plan:

- Use a fresh clone, preferably `git clone --no-local` for local-source clones.
- Run `git filter-repo --analyze` first.
- Decide whether to push to a new repo or coordinate a force-push.
- Expect commit IDs and tags to change.
- Coordinate all other clones to avoid reintroducing old history.
- Do not run `--force` in this working tree without explicit user approval.

## Self-Review

- Spec coverage: covers current-tree cleanup, `docs/` preservation, `dev-docs/` non-touch rule, `.ai/` historical preservation, and history rewrite deferral.
- Placeholder scan: no unresolved placeholder markers remain.
- Scope check: implementation is split by active root identity, config, Flux, docs, local skills, validation, and report. Git history rewriting is separated.
