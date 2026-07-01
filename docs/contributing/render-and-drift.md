---
title: Render and Drift
description: How the render/drift invariant works, how to run the render locally, and what to do when drift is detected.
type: guide
audience: [contributor, operator]
tags: [config, ci, drift]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - ../adr/0011-config-compiler.md
  - ../concepts/config-compiler.md
---

# Render and Drift

The config compiler (ADR-011) enforces a core invariant:

> **`render(inputs) == committed tree`**

Every generated file in the repo must match what the render playbook would produce from the current inputs. Hand-editing generated files is not permitted. If the committed tree diverges from the rendered output, that is **drift**, and it must be resolved before merging.

---

## What the invariant means

The render playbook (`render-config.yml`) reads two input layers:

- `config/defaults/` — template-owned defaults; never edit these directly.
- `config/clusters/<cluster>/` — your cluster-specific overrides (`cluster.yaml`, `nodes.yaml`, Talos overlays).

It merges the two layers and emits generated artifacts into:

- `infra/ansible/inventories/*/group_vars/` — generated Ansible group vars
- `infra/talos/patches/generated/` — generated Talos machine-config patches

These generated files are committed to git so the repo is always self-contained, but they are **not** a source of truth. The inputs in `config/` are the source of truth. Do not edit generated files by hand; edit the inputs and re-render.

---

## Run the render locally

From the repo root, run:

```bash
ansible-playbook infra/ansible/playbooks/render-config.yml
```

To target a specific cluster (other than the default `example`):

```bash
ansible-playbook infra/ansible/playbooks/render-config.yml -e config_cluster=<cluster-name>
```

The playbook runs on `localhost`, gathers no facts, and makes no network calls — it only reads and writes files within the repository tree.

---

## Check for drift

To detect drift without writing files, use check mode. The playbook renders to a temporary directory and diffs the output against the committed files:

```bash
ansible-playbook infra/ansible/playbooks/render-config.yml -e config_render_check=true
```

Or for a specific cluster:

```bash
ansible-playbook infra/ansible/playbooks/render-config.yml \
  -e config_cluster=<cluster-name> \
  -e config_render_check=true
```

If the playbook reports differences, drift exists. The output shows which generated files would change.

You can also check drift manually with `git diff` after a render:

```bash
ansible-playbook infra/ansible/playbooks/render-config.yml
git diff
```

A clean diff means no drift.

---

## CI enforcement

> **Note:** CI drift checking is not yet configured. When a CI pipeline is added, it will run the render in check mode (`config_render_check=true`) on every pull request and fail if the rendered output differs from the committed tree.

---

## When drift happens

Drift occurs when:

- A contributor edits a generated file directly instead of editing the inputs and re-rendering.
- A template update changes `config/defaults/` but the generated files have not been re-rendered.
- A merge introduces changes to both inputs and generated files that are inconsistent.

**To fix drift:**

1. Identify the source of the divergence (check `git log` on the affected inputs and generated files).
2. Edit the inputs (`config/clusters/<cluster>/cluster.yaml`, `nodes.yaml`, or Talos overlays) to capture the intended change.
3. Re-render:

   ```bash
   ansible-playbook infra/ansible/playbooks/render-config.yml -e config_cluster=<cluster-name>
   ```

4. Verify no remaining drift:

   ```bash
   ansible-playbook infra/ansible/playbooks/render-config.yml \
     -e config_cluster=<cluster-name> \
     -e config_render_check=true
   ```

5. Commit the updated inputs and regenerated files together.

> **Warning:** Never commit only the generated files without also committing the input change that produced them. The invariant requires inputs and generated files to be consistent in every commit.

---

## What requires a render

Not all changes require a render. Use this as a guide:

| Change type | Requires render? |
|---|---|
| Enable or disable a feature in `cluster.yaml` | Yes |
| Change CNI, disk/volumes, network, podCIDR | Yes |
| Add or remove a Talos system extension | Yes |
| Patch an enabled feature's Flux overlay | No |
| Add or update a Flux app | No |
| Rotate a SOPS secret | No |
| Update `config/defaults/` (template update) | Yes |

See [ADR-011](../adr/0011-config-compiler.md) for the full decision record.
