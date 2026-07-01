# Template identity cleanup closing report

## What changed

Removed active `kube1` identity from the current tree. The repo now reads as a reusable Talos Kubernetes deployment template using `example` as the concrete cluster name and `<cluster>` in prose.

### Root identity (3 files)
- `README.md` — title from `# kube1` to `# Talos Kubernetes Bootstrap Template`, summary neutralized
- `AGENTS.md` — identity line changed to "reusable Talos Linux Kubernetes cluster deployment template"
- `opencode.jsonc` — stale `~/Projects/infra/kube1/**` external dir replaced with current repo path

### Config cluster (7+ files)
- `config/clusters/kube1/` renamed to `config/clusters/example/`
- `config/clusters/example/cluster.yaml` — `cluster.name: example`, hostnames `example-hb*`, network/firewall/SSH key names → `example`
- `config/clusters/example/nodes.yaml` — comments and examples updated
- `config/clusters/example/talos/*.yaml` — comments neutralized
- `config/schemas/cluster-input.schema.json` — `$id` → `talos-bootstrap-template.local`, `title` → `Cluster input`
- `config/schemas/nodes.schema.json` — `$id` → `talos-bootstrap-template.local`, `title` → `Per-node overrides`

### Ansible/infra (17 files)
- `config_render/defaults/main.yml` — default `config_cluster: example`
- Generated group_vars — `cluster_name: example`, `flux_path: ./flux/clusters/example`, hcloud label/network → `example`
- Static inventories — hostnames `example-hb1/2/3`
- Task comments and playbook help text updated

### Flux (15 files)
- `flux/clusters/kube1/` renamed to `flux/clusters/example/`
- `flux/infrastructure/{controllers,configs}/overlays/clusters/kube1/` renamed to `example/`
- All overlay references in kustomization.yaml resources updated
- `flux/apps/README.md` — paths use `<cluster>`, concrete examples use `example`
- `flux/clusters/example/flux-system/gotk-sync.yaml` — path updated (repo URL kept as-is)
- Component kustomization.yaml comments updated

### Docs (6 files)
- `docs/node-roles.md`, `docs/upgrades.md`, `docs/adding-a-provider.md` — paths use `<cluster>`, hostname examples use `example-hb*`
- `docs/cert-manager-ingress.md` — title and prose neutralized, paths parameterized
- `docs/openebs-local-storage.md` — title and prose neutralized, paths parameterized
- `docs/cluster-bootstrap/bootstrap-recovery.md` — prose updated

### Agent identity statements (9 files)
- `.opencode/agents/*.md` — "You are the kube1 X" → "You are the X" in all 9 agent files
- `.opencode/skills/kube1-orchestration/SKILL.md` — one `explore` agent rule neutralized (matching AGENTS.md update)

### Intentionally left unchanged
- `docs/kube1/adrs/` — historical kube1-specific ADRs preserved as-is
- `docs/cluster-bootstrap/adrs/` — historical ADR content preserved (examples are historical context, per user direction)
- `.ai/` historical plans/reports — not rewritten
- `config/clusters/example/cluster.yaml` `repoUrl` and `repository` — actual GitHub repo name kept
- Actual IPs, server types, provider config in `cluster.yaml` — concrete example data preserved
- Skill names (`kube1-*`) and agent descriptions — metadata, deferred to separate OpenCode config task

## What was validated

- `ansible-lint` — 0 failures, 0 warnings (62 files)
- `kustomize build flux/infrastructure/controllers` — PASS
- `kustomize build flux/infrastructure/configs` — PASS (3 resources: 2 ClusterIssuers + GatewayClass)
- `kustomize build flux/clusters/example` — expected failure (no kustomization.yaml, documented auto-swept directory)
- `ansible-playbook playbooks/render-config.yml -e config_cluster=example -e config_render_check=true` — PASS, "No render drift detected"
- `rg` search: no `kube1` in root, config, flux, or docs files outside intentionally-preserved areas
- `rg` search: no `dev-docs/` references

## What was explicitly not run

- No live cluster commands, provider-costing, SOPS decryption, `talosctl apply-*`, `kubectl`, or `flux reconcile`
- No git history rewriting
- No skill renames (deferred to separate `customize-opencode` task)

## Remaining kube1 references

All remaining `kube1` matches in the active tree are in `.opencode/` skill/agent metadata:
- Skill names (`kube1-orchestration`, `kube1-workflow`, `kube1-architecture`, `kube1-validation`) — intentionally deferred
- Agent descriptions (line 2 metadata) — intentionally deferred
- `adr-writer.md:52` and `kube1-architecture/SKILL.md:29` — valid path references to existing `docs/kube1/adrs/`

## Follow-ups / risks

1. **Rename local OpenCode skills.** Skill names (`kube1-*`) should be renamed using the `customize-opencode` skill in a separate task. Skill renames affect agent routing and should not be done mechanically.
2. **Verify `.ai/plans/2026-06-30-app-entrypoints.md`.** This untracked plan contains `kube1` references. User should decide whether to update, keep, or delete it.
3. **History rewrite is deferred.** If full commit-history cleanup is ever desired, use a separate `git-filter-repo` plan with a fresh clone and explicit approval for force-push.
