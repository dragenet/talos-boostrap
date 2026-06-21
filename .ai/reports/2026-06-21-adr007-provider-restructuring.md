# ADR-007 provider restructuring — closing report

**Date:** 2026-06-21
**Status update:** 2026-06-22 — reviewer findings resolved in a follow-up slice; next refactor slice is ADR-011 config-compiler foundation (see Follow-ups).
**Type:** Pure refactor — no behavior change
**Plan:** `.ai/plans/2026-06-21-adr007-provider-restructuring.md`

## What changed

Moved all provider-specific code under uniform `providers/<name>/` directories to enforce ADR-007's "new provider = add `providers/<name>/` only" rule across inventories, playbooks, roles, and config.

**Directory moves:**

- `inventories/hcloud/` → `inventories/providers/hcloud/`
- `inventories/manual/` → `inventories/providers/manual/`
- `playbooks/hcloud/` → `playbooks/providers/hcloud/`
- `roles/hcloud_servers/` → `roles/providers/hcloud/hcloud_servers/`

**Config:**

- `ansible.cfg` `roles_path` updated to `roles:roles/providers/hcloud` so the `hcloud_servers` role remains discoverable.
- SOPS lookup path in `hcloud.yml` updated to `inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml`.

**Removed:**

- `playbooks/providers/hcloud/utils/` (4 files: `open-talos-api.yml`, `close-talos-api.yml`, `open-k8s-api.yml`, `close-k8s-api.yml`) — redundant with the `hcloud_firewall_open_access` flag in `group_vars`.

**Reference updates:**

- Comments in playbooks and roles.
- Docs: `CLAUDE.md` (including the lock-out recovery section, rewritten to use the `hcloud_firewall_open_access` flag), `adding-a-provider.md`, `upgrades.md`, `node-roles.md`, `ADR-001`.
- `.claude/settings.local.json`.

## What was validated

- `ansible-lint`: 0 failures, 0 warnings, 29 files processed.
- Grep sweep: no stale references to the old paths remain.
- Variable names (`hcloud_servers_*`) unchanged — only filesystem paths moved.
- Playbooks still target the current live 3-node hcloud cluster.

## What was not run

- `ansible-playbook` against the live cluster. Per repo policy the user runs playbooks in their own terminal; this session did not exercise the moved playbooks. Risk is low because the refactor is path-only and `ansible-lint` was clean.

## Reviewer findings (resolved 2026-06-22)

`ansible-reviewer` raised 5 issues against this slice. All 5 were addressed in a follow-up at `.ai/plans/2026-06-21-adr007-reviewer-fixes.md` and reported at `.ai/reports/2026-06-21-adr007-reviewer-fixes.md`:

1. **Medium** — `ansible.cfg` `roles_path` is now `roles:roles/providers`; the role is referenced as `hcloud/hcloud_servers`. Global config is genuinely provider-agnostic and adding a second provider no longer requires a global config change.
2. **Low** — `manual/hosts.yml` stale `-i inventories/manual` comment updated.
3. **Low** — `hcloud/hcloud.yml` stale `hosts.yml` reference removed.
4. **Low** — `adding-a-provider.md` `.sops.yaml` step corrected; the existing generic regex already covers new providers.
5. **Low** — `CLAUDE.md` tree `utils/` entries removed.

## Follow-ups

- **Next refactor slice: ADR-011 config-compiler foundation** — plan at `.ai/plans/2026-06-22-next-refactor-slice.md`. Scope is intentionally narrow and low-risk: Ansible-side schema/input scaffold plus dry-run-safe rendering. **Flux bootstrap, Cilium installation, the `_components/` catalog, and any Hetzner / Talos / Flux mutation are explicitly out of scope** for this slice and remain separate work. Validation is limited to `ansible-lint` and `ansible-playbook --check --diff` against the manual provider — no `hcloud`, `talosctl`, `kubectl`, or `flux reconcile` is run by agents.
- **Prior ADR-007 reviewer fixes** are resolved (see above) — no carry-over from this refactor.

## Risks

- **Low overall.** Pure refactor; variable names and behavior unchanged. The prior `roles_path` architectural risk (#1) is now resolved, and the `providers/<name>/` contract is genuinely provider-agnostic. Remaining repo-wide risk is the usual "not yet built" backlog (Flux bootstrap, Cilium, config catalog), tracked separately in the next-refactor-slice plan.
