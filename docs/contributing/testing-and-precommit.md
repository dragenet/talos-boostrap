---
title: Testing and Pre-commit
description: Pre-commit hooks configured in this repo, how to run them, and the manual testing checklist for contributors.
type: guide
audience: [contributor]
tags: [testing, precommit, ci]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - development-setup.md
  - render-and-drift.md
---

# Testing and Pre-commit

This guide covers the pre-commit hooks configured in this repo and the manual checks to run before submitting a change.

---

## Install pre-commit

If you have not already done so, install the hooks:

```bash
pre-commit install
```

Once installed, the hooks run automatically on every `git commit`. See [Development Setup](development-setup.md) for full prerequisite instructions.

---

## Configured hooks

The following hooks are defined in `.pre-commit-config.yaml`:

| Hook ID | Source | What it checks |
|---|---|---|
| `ansible-lint` | `ansible/ansible-lint` v26.4.0 | Ansible best practices and correctness for files under `infra/ansible/` |

> **Note:** The hook is scoped to `infra/ansible/` (`files: ^infra/ansible/`). Changes outside that directory do not trigger `ansible-lint`.

---

## Run all hooks

To run every configured hook against all files in the repo:

```bash
pre-commit run --all-files
```

This is the authoritative check. Run it before opening a pull request.

---

## Run a specific hook

To run only one hook:

```bash
pre-commit run ansible-lint
```

Replace `ansible-lint` with the `id` of whichever hook you want to target. You can also limit the hook to specific files:

```bash
pre-commit run ansible-lint --files infra/ansible/playbooks/render-config.yml
```

---

## Manual testing checklist

After making changes, work through this checklist before committing:

### Config changes

- [ ] Run the render playbook and confirm no unexpected diff:

  ```bash
  ansible-playbook infra/ansible/playbooks/render-config.yml -e config_cluster=<cluster-name>
  git diff
  ```

- [ ] Run render in check mode to confirm no drift remains:

  ```bash
  ansible-playbook infra/ansible/playbooks/render-config.yml \
    -e config_cluster=<cluster-name> \
    -e config_render_check=true
  ```

See [Render and Drift](render-and-drift.md) for details.

### Ansible changes

- [ ] `ansible-lint` passes:

  ```bash
  pre-commit run ansible-lint --all-files
  ```

### All changes

- [ ] Full pre-commit suite passes:

  ```bash
  pre-commit run --all-files
  ```

---

## Schema validation

The repo ships JSON schemas for cluster input files:

- `config/schemas/cluster-input.schema.json` — validates `cluster.yaml`
- `config/schemas/nodes.schema.json` — validates `nodes.yaml`

You can validate your cluster inputs against the schema with any JSON Schema validator. For example, using `check-jsonschema` (install via `pip install check-jsonschema`):

```bash
check-jsonschema --schemafile config/schemas/cluster-input.schema.json \
  config/clusters/<cluster-name>/cluster.yaml

check-jsonschema --schemafile config/schemas/nodes.schema.json \
  config/clusters/<cluster-name>/nodes.yaml
```

Schema validation is not yet wired into the pre-commit pipeline; run it manually when editing `cluster.yaml` or `nodes.yaml`.

---

## Skipping hooks

You can bypass hooks in exceptional cases with:

```bash
git commit --no-verify
```

> **Warning:** Only skip hooks when you have a specific, documented reason (e.g. committing a work-in-progress branch that is not yet ready for full validation). Never skip hooks on a final commit destined for the main branch.
