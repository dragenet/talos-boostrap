# Universal Flux bootstrap authentication — closing report

**Date:** 2026-06-24
**Plan:** `.ai/plans/2026-06-24-universal-flux-bootstrap-auth.md`

Implementation and review are complete. No live cluster, provider, or git-host actions were run by agents.

## What changed

- Added `flux.provider` and `flux.auth.*` schema/defaults.
- Rendered non-secret Flux auth vars into generated common group vars.
- Split `flux_bootstrap` into early pre-Flux checks and late Flux-only auth checks.
- Rewrote Flux bootstrap dispatch for GitHub, Bitbucket Server/DC, and generic Git.
- Added/updated ADR-015 for the durable auth contract and matrix.

## Validation

- `ansible-lint` passed.
- `ansible-playbook playbooks/render-config.yml -e config_render_check=true` passed with no drift.
- `ansible-playbook --syntax-check` passed for both hcloud and manual inventories.
- Ansible reviewer approved the final state.

## Remaining user steps

- Fill in `config/overrides/cluster.yaml` with the chosen `flux.provider` / `flux.auth.*` values.
- Store `flux_auth_token` and optional SSH passphrase in SOPS-backed inventory vars.
- Run `render-config.yml` and then `bootstrap-flux.yml` from the user's terminal.

## Follow-ups

- Write the user-facing Flux auth runbook.
- Run live validation once a provider/mode is chosen.
