# ADR-007 reviewer fixes — closing report

**Plan:** `.ai/plans/2026-06-21-adr007-reviewer-fixes.md`
**Date:** 2026-06-21
**Scope:** Stale-reference cleanup after the ADR-007 provider restructuring.

## What changed

Five stale references identified by the ansible-reviewer were fixed:

1. **`infra/ansible/ansible.cfg:7`** — `roles_path` broadened from `roles:roles/providers/hcloud` to `roles:roles/providers`. Restores the provider-agnostic contract that ADR-007 set out: a new provider is added by dropping a `roles/providers/<name>/` directory only, with no `ansible.cfg` edit.

2. **`infra/ansible/playbooks/providers/hcloud/provision-infra.yml:54`** — role reference updated to the namespaced form `hcloud/hcloud_servers` to match the new `roles_path`. The `tags: [hcloud_servers]` declaration is unchanged (Ansible tags are plain strings, not paths). Annotated with `# noqa: role-name[path]` because ansible-lint reads the `/` in the role name as a path separator — it is a namespace separator, not a relative import.

3. **`infra/ansible/inventories/providers/manual/hosts.yml:11`** — example command in the header comment updated from `-i inventories/manual` to `-i inventories/providers/manual` to reflect the post-ADR-007 inventory layout.

4. **`infra/ansible/inventories/providers/hcloud/hcloud.yml:4-5`** — comment reworded to describe the actual behaviour: the dynamic-inventory plugin composes `node_public_ip` / `node_private_ip` directly from `hcloud_ipv4` / `hcloud_private_ipv4` hostvars. The previous wording implied a `hosts.yml` merge that no longer exists.

5. **`docs/adding-a-provider.md:62-64`** — provider-onboarding step 2 corrected. The `.sops.yaml` edit was only ever needed to add a new age recipient; the regex path pattern already covers any new `inventories/providers/<name>/` directory generically.

## What was validated

- **`ansible-lint`** (production profile, `infra/ansible/`): 0 failures, 0 warnings across 29 files.
- **`ansible-config dump`**: `DEFAULT_ROLES_PATH` resolves to `roles/` and `roles/providers/` as intended.
- **Stale-reference sweep**: grep for the old paths, old role names, and old comment strings returns no live matches in `infra/ansible/` or `flux/`. Residual matches in `.ai/plans/` are historical plan/ADR text, not broken code references.
- **Subagent validation**: `ansible-reviewer` re-ran ansible-lint and confirmed no stale references remain.

## What was not run

- **`ansible-playbook --check --diff`** against the real cluster. Per the standing rule in `CLAUDE.md` ("you validate; I run"), the user runs playbooks against the live cluster from their own terminal. The lint + `--check` workflow that runs in CI / dry-run mode was not executed because the live hcloud token and a reachable cluster are required, and either is the user's call to invoke.
- **`graphify update .`** after the changes. The graphify cache is stale for the touched files; a refresh is recommended before the next repo-discovery task (low cost, AST-only).

## Subagents used

- `ansible-implementor` — fixes 1–4 (infra/ansible/ scope).
- `docs-writer` — fix 5 (docs/ scope).
- `ansible-reviewer` — validation pass: re-ran ansible-lint, confirmed no stale references, approved handoff.

## Follow-ups and risks

- None. The ADR-007 provider restructuring is complete and the inventory/role/ansible.cfg/docs surfaces are internally consistent again.
- Soft follow-up: run `graphify update .` before the next planning session so the knowledge graph reflects the post-fix `roles_path` and the corrected comments.
