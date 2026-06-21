# 2026-06-22 — Next refactor slice after ADR-007

## Goal and scope

Identify and execute the next safe, coherent one-session slice after ADR-007 provider restructuring.

Recommended slice: **ADR-011 config-compiler foundation, with status/report reconciliation first**.

This is intentionally a **low-risk Ansible/docs slice**, not a live-cluster or Flux bootstrap slice. It should establish the compiler contract and validation path without changing running infrastructure, creating Hetzner resources, applying Talos configs, or reconciling Flux.

## Current-state facts from repo context

- ADR-007 provider restructuring is complete and reviewer follow-up fixes appear complete in current repo state.
- `.ai/reports/2026-06-21-adr007-reviewer-fixes.md` reports the 5 post-review findings fixed, including provider role discovery and stale docs comments.
- ADR-010/011/012/013 describe the next architecture layer:
  - ADR-010: Ansible owns pre-Flux/bootstrap handoff; Flux owns ongoing cluster resources.
  - ADR-011: a config compiler should render deterministic artifacts from `cluster.yaml` plus overlays, with one writer per file.
  - ADR-012: feature catalog under `_components/`, toggled by config.
  - ADR-013: Talos config layering uses structured helpers plus raw patch fragments.
- Config compiler implementation is not present yet: no `config/` input tree, no `configure.yml` render entrypoint, no compiler role, and no generated Flux/component catalog implementation.
- Flux remains unbootstrapped and `flux/` content remains placeholders. This plan does not change that.
- `CLAUDE.md` status is broadly accurate that Flux/Cilium/config catalog are not built. It may need a small status note only if the implementation changes repository capabilities.

## Official docs to consult before implementation

Implementors must consult these official docs before making config/compiler changes:

- Ansible roles and reuse: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
- Ansible role argument validation: same page, `meta/argument_specs.yml` section.
- Ansible inventory and `group_vars`: https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html
- Ansible variable precedence: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable
- Ansible check/diff mode: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html
- ansible-lint docs/rules: https://ansible.readthedocs.io/projects/lint/
- Talos patching: https://www.talos.dev/latest/talos-guides/configuration/patching/
- Talos machine configuration reference: https://www.talos.dev/latest/reference/configuration/
- Talos reproducible machine configuration: https://www.talos.dev/v1.13/configure-your-talos-cluster/system-configuration/reproducible-machine-configuration/

Flux, Cilium, and Hetzner docs are only required if the slice expands to Flux bootstrap, CNI installation, or provider-specific resource changes. This recommended slice should not.

## Ordered implementation steps

1. **Preflight status reconciliation** — owner: `repo-fast-context`
   - Re-read current ADR-007 reports, ADR-010/011/012/013, `CLAUDE.md`, and `docs/adding-a-provider.md`.
   - Confirm there are no remaining ADR-007 reviewer fixes in code before starting new compiler work.
   - Identify any stale status text that must be updated after the compiler scaffold lands.

2. **Define the narrow compiler foundation contract** — owner: live orchestrator + user decision if needed
   - Keep the slice limited to schema/input/scaffold and dry-run-safe rendering.
   - Do not silently choose a durable output policy if ADR-011 is ambiguous.
   - If the exact config input path or generated-output ownership differs from ADR-011, pause for user agreement and mark an ADR-011 update.

3. **Implement Ansible-lane compiler scaffold** — owner: `ansible-implementor`
   - Add the minimal Ansible entrypoint/role needed for config compilation, following existing repo conventions.
   - Add role argument specs for public variables.
   - Add a non-secret example config input, not real credentials or rendered Talos secrets.
   - Prefer validation and deterministic preview output over mutating existing `group_vars`, Talos patches, or Flux resources in this first slice.
   - Ensure all generated or preview output paths are git-safe and do not clobber hand-authored files.

4. **Flux lane** — owner: none for this slice
   - Do not implement Flux bootstrap, Cilium, HelmReleases, or `_components/` catalog resources in this slice.
   - If any Flux files are touched accidentally, split the work and route to `k8s-implementor` with separate validation.

5. **Docs/status update if needed** — owner: `docs-writer` for `docs/`; live orchestrator/user-approved edit for `CLAUDE.md` if required
   - If compiler scaffold adds a new user workflow, document how to run it and what it does not do yet.
   - Keep `CLAUDE.md` implementation status honest: config compiler scaffold exists, full Flux/Cilium/component catalog still not built.
   - If no user-visible workflow lands, skip docs beyond the closing report.

6. **Review Ansible lane** — owner: `ansible-reviewer`
   - Review idempotency, FQCN usage, variable namespacing, check-mode behavior, secret handling, and generated-output safety.
   - Confirm no live-cluster, Hetzner, Talos, or Flux operations are embedded in validation-only compiler steps.

7. **Closing report** — owner: `report-writer`
   - Write `.ai/reports/2026-06-22-next-refactor-slice.md` or a slug matching the actual implementation.
   - Record what changed, validation results, remaining compiler phases, and whether Flux bootstrap remains separate.

## Validation required before handoff

From `infra/ansible/`:

```bash
ansible-lint
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/configure.yml --check --diff
```

If the entrypoint lands outside `playbooks/configure.yml`, use the actual path chosen by the implementation and report it.

Optional/non-mutating checks if applicable:

```bash
ansible-inventory -i inventories/common -i inventories/providers/manual --list
ansible-doc -t role <new-role-name>
```

No `ansible-playbook` against the hcloud provider, no `hcloud`, no `talosctl`, no `kubectl`, and no `flux reconcile` should be run by agents for this slice without explicit user approval.

If Flux files are touched despite the recommended scope, add Flux/K8s validation before handoff:

```bash
kustomize build flux/clusters/kube1
kustomize build flux/infrastructure/controllers
kustomize build flux/infrastructure/configs
kustomize build flux/apps
```

## Risks, side effects, and live-cluster boundaries

- **Main risk:** premature compiler output could overwrite hand-authored inventory, Talos patch, or Flux files. Keep first slice to validation/scaffold or isolated preview output unless the user approves a writer policy.
- **Secrets risk:** never render or commit Talos secrets, kubeconfigs, decrypted SOPS data, or API tokens.
- **Check-mode risk:** tasks that read local files may need `check_mode: false`; only use it for read-only tasks and explain why inline.
- **Architecture risk:** ADR-011/013 layering must remain deterministic. Later layers must have explicit precedence.
- **Live-cluster boundary:** this slice must not apply Talos configs, bootstrap Flux, install Cilium, call Hetzner APIs, or change running cluster resources.
- **Cost boundary:** no cost-incurring Hetzner operations are expected.

## ADR needs

- No new ADR is required if implementation stays within ADR-011/012/013.
- If the team decides a durable compiler output location, generated-file commit policy, or ownership rule not already specified by ADR-011, add a decision step and assign `adr-writer` to update/write the ADR after agreement.
