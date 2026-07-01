---
title: Architecture Overview
description: High-level map of the three subsystems — Ansible, config compiler, and Flux — and how data flows between them.
type: concept
audience: [user, contributor]
tags: [architecture, talos, flux, ansible]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - config-compiler.md
  - layered-talos-config.md
  - provider-portability.md
  - gitops-model.md
---

# Architecture Overview

This template provides a reproducible, provider-portable stack for deploying a Talos Linux Kubernetes cluster. It is built from three cooperating subsystems: the **config compiler**, the **Ansible provisioning and bootstrap layer**, and the **Flux GitOps engine**. Each subsystem has a distinct responsibility; together they cover the full cluster lifecycle from first boot to ongoing day-2 operations.

## The Three Subsystems

### 1. Config Compiler

The config compiler transforms a small, centralized set of user-owned inputs into all the artifacts the cluster needs: Talos machine-config patches, Ansible `group_vars`, and Flux kustomization selections.

Inputs live under `config/`:

- `config/defaults/` — template-owned defaults; updated when you pull upstream improvements.
- `config/clusters/<cluster>/` — your cluster-specific overrides (`cluster.yaml`, `nodes.yaml`, Talos overlay patches).
- `config/schemas/` — validation schemas applied during render.
- `config/talos/` — shared, template-owned Talos base configs and component fragments.

The render playbook (`infra/ansible/playbooks/render-config.yml`) merges these layers and emits generated artifacts that are committed to git. CI asserts that `render(inputs) == committed tree`; any uncommitted hand-edit fails the check.

See [Config Compiler](config-compiler.md) and [Layered Talos Config](layered-talos-config.md) for details.

### 2. Ansible Provisioning and Bootstrap Layer

Ansible is the execution engine for everything that must happen before or during Flux's first reconcile. It reads the committed `config/` tree (after rendering) and drives:

- `render-config.yml` — runs the config compiler.
- `bootstrap-talos.yml` — applies Talos machine configs, bootstraps etcd.
- `bootstrap-flux.yml` — installs pre-Flux components (CNI, provider CCM), then runs `flux bootstrap`.
- `upgrade-talos.yml` — upgrades Talos on existing nodes.

Provider-specific provisioning playbooks live under `infra/ansible/playbooks/providers/<provider>/`. All other playbooks are provider-agnostic.

See [Provider Portability](provider-portability.md) for how the provider abstraction works.

### 3. Flux GitOps Engine

After `flux bootstrap` runs, Flux owns ongoing cluster state. It watches the `flux/` tree in this git repository and reconciles `Kustomization` and `HelmRelease` objects in dependency order:

```
infrastructure/controllers → infrastructure/configs → apps
```

Features — cert-manager, ingress controllers, OpenEBS, and others — are catalog entries under `flux/infrastructure/controllers/_components/`. Enabling a feature is structural: the render adds it to the generated `selected/` kustomization; Flux then reconciles it automatically.

See [GitOps Model](gitops-model.md) for the full picture.

## Data Flow

The high-level sequence from user input to running cluster:

```
config/clusters/<cluster>/cluster.yaml
config/clusters/<cluster>/nodes.yaml
config/defaults/
        │
        ▼  render-config.yml
        │
        ├─▶ group_vars/        (consumed by Ansible roles)
        ├─▶ Talos patches       (consumed by bootstrap-talos.yml)
        └─▶ flux/…/generated/  (consumed by Flux after bootstrap)
        │
        ▼  bootstrap-talos.yml
        │
        ├─▶ Talos machine configs applied to nodes
        └─▶ etcd bootstrapped
        │
        ▼  bootstrap-flux.yml
        │
        ├─▶ Pre-Flux components installed (CNI, CCM)
        └─▶ flux bootstrap → Flux reconciles git → cluster reaches desired state
```

After bootstrap, the day-2 loop is purely git-driven: commit a change to `flux/`, and Flux reconciles it within its configured interval. Talos machine-config changes (disk layout, network, image extensions) require re-running the render and bootstrap playbooks, because those changes cannot be expressed in git alone.

## What Lives Where

| Tree | Owner | Purpose |
|---|---|---|
| `config/defaults/` | Template | Shared defaults; pulled from upstream |
| `config/clusters/<cluster>/` | User | Per-cluster overrides; never overwritten by upstream |
| `config/schemas/` | Template | Render-time validation |
| `config/talos/` | Template | Shared Talos base configs and components |
| `infra/ansible/` | Template | Playbooks and roles; provider code under `providers/` |
| `flux/` | User + Render | GitOps tree; `_components/` is template-owned |

## Further Reading

- [Config Compiler](config-compiler.md) — how inputs become Talos and Flux artifacts
- [Layered Talos Config](layered-talos-config.md) — Talos patch merge order
- [Provider Portability](provider-portability.md) — how provider differences are isolated
- [GitOps Model](gitops-model.md) — Flux feature catalog, overlay ownership, and day-2 operations
