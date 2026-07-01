---
title: Config Compiler
description: How the render playbook merges defaults and cluster overrides into deterministic Talos, Ansible, and Flux artifacts.
type: concept
audience: [user, contributor]
tags: [config, talos, compiler]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - architecture-overview.md
  - layered-talos-config.md
  - ../adr/0011-config-compiler.md
---

# Config Compiler

The config compiler turns a small, centralized set of human-authored inputs into all the generated artifacts the cluster needs — Talos machine-config patches, Ansible `group_vars`, and Flux kustomization selections. Its core promise: one render pass produces a deterministic, fully-committed output tree; no value needs to be repeated across multiple files.

## Problem It Solves

Without a compile step, cluster configuration scatters across Ansible `group_vars`, Talos patch files, and Flux manifests. Updating a single value — for example, a pod CIDR — requires editing multiple files in multiple formats, with no enforcement that they stay consistent.

The config compiler provides a single declarative input surface. You declare your cluster's shape once; the render produces all downstream artifacts from it.

## Inputs: Two-Layer Overlay

```
config/
  defaults/                    # TEMPLATE-OWNED — upstream improvements land here
  clusters/<cluster>/          # USER-OWNED — only what differs from defaults
    cluster.yaml               # feature flags, network, CNI, provider selection
    nodes.yaml                 # per-node hardware: IPs, roles, disk selectors
    talos/                     # user-authored Talos overlay patches
  schemas/                     # validation rules applied during render
  talos/                       # template-owned shared Talos base configs
```

The render computes `deep_merge(defaults, clusters/<cluster>)`, with the cluster layer winning. Merge semantics differ by key type:

- **Merge** — maps such as `features` are merged key-by-key.
- **Replace** — user-owned lists such as `nodes` and `storage.volumes` replace the default entirely.
- **Compute** — union keys such as `talos.extensions` are assembled from base + feature-implied extensions + any `talos.extraExtensions` you declare. The render rejects a direct override of a compute key; use the `*Extra*` variant instead.

## What the Render Emits

Running `infra/ansible/playbooks/render-config.yml` produces:

| Output | Consumer |
|---|---|
| `group_vars/` — flat, namespaced vars | Ansible roles (`cluster_*`, `talos_*`, etc.) |
| Talos machine-config patches | `bootstrap-talos.yml` via `talosctl` `--config-patch` |
| `flux/…/generated/selected/kustomization.yaml` | Flux after `flux bootstrap` |
| `cluster-vars` ConfigMap | Flux `postBuild.substituteFrom` for scalar substitution |

All generated files are marked *generated, do not edit*. They are committed to git so the repo is always self-describing, but they are not the source of truth — the inputs are.

## The Drift Invariant

Rendered output is committed, but CI asserts:

```
render(inputs) == committed tree
```

Any uncommitted hand-edit or stale generated file fails this check. This catches drift before it reaches the cluster and prevents PRs that edit generated files directly.

## Ownership Contract

Every file belongs to exactly one owner; the other two never touch it.

| Owner | Files |
|---|---|
| **Template** | `config/defaults/`, component bases, playbooks and roles |
| **Render** | `group_vars/`, Talos patches, Flux `generated/`, `cluster-vars` |
| **User** | `config/clusters/<cluster>/`, Flux overlay `patches/`, SOPS secrets |

Because template updates touch only template-owned files, `git pull upstream` is conflict-free against your cluster-specific config.

## Customization Ladder

ADR-011 defines three tiers of customization:

1. **Use as-is** — enable a feature in `cluster.yaml`, run the render.
2. **Tweak** — keep the feature enabled; add a Kustomize patch in your user overlay. This is a git-only change; Flux reconciles it automatically.
3. **Replace** — disable the feature in `cluster.yaml` (the render drops it from `generated/selected/`); deploy your own version via Flux.

## When to Re-Run the Render

Changes to `cluster.yaml` or `nodes.yaml` require re-running the render before they take effect. Some changes also require re-running `bootstrap-talos.yml`:

- Feature enable/disable
- Talos machine-config changes: CNI, disk/volume layout, network, Talos image extensions, kubelet arguments

Pure-git, no-render changes include: patching enabled features in the Flux overlay, adding or updating apps, rotating SOPS secrets, and any Flux-only scalar substitution values.

## Key Directories

| Path | Description |
|---|---|
| `config/defaults/` | Template-owned defaults; never edited by users |
| `config/clusters/<cluster>/` | User-owned cluster config |
| `config/schemas/` | Render-time validation schemas |
| `config/talos/` | Shared Talos base configs and component fragments |

## Further Reading

- [Architecture Overview](architecture-overview.md) — where the config compiler fits in the overall system
- [Layered Talos Config](layered-talos-config.md) — how Talos patches are assembled from the compiler's output
- [ADR-011: Config Compiler](../adr/0011-config-compiler.md)
