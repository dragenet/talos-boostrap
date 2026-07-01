---
title: Concepts
description: Explanations of how the Talos Kubernetes bootstrap template works — architecture, config compilation, GitOps, and provider portability.
type: index
audience: [user, operator, ai]
tags: [concepts, talos, kubernetes, architecture]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - ../index.md
  - ../guides/index.md
---

# Concepts

Explanations of the systems and models that underpin this template. Each concept answers "what is X and why does it work this way?" Use these to build understanding before or alongside the task guides.

## Pages

| Concept | Description |
|---|---|
| [Architecture Overview](architecture-overview.md) | High-level map of all components and how they fit together |
| [Config Compiler](config-compiler.md) | How the compiler renders per-cluster Talos machine configs from a shared defaults layer |
| [Layered Talos Config](layered-talos-config.md) | The merge order for base, role, provider, and cluster-specific patches |
| [Provider Portability](provider-portability.md) | How the template abstracts provider differences behind an inventory contract |
| [GitOps Model](gitops-model.md) | The Flux-driven GitOps workflow and the render/drift invariant |

## Descriptions

**[Architecture Overview](architecture-overview.md)** — The three subsystems (config compiler, Ansible, Flux) and the data flow from `cluster.yaml` to a running cluster.

**[Config Compiler](config-compiler.md)** — How `render-config.yml` merges `config/defaults/` with `config/clusters/<cluster>/` to produce deterministic Talos, Ansible, and Flux artifacts. Covers the drift invariant enforced by CI.

**[Layered Talos Config](layered-talos-config.md)** — The two-layer Talos machine-config composition model: structured helpers in `cluster.yaml` for the common case, plus raw Talos patch fragments for anything else.

**[Provider Portability](provider-portability.md)** — The `providers/<name>/` rule, the standardized Ansible inventory hostvar contract, the dynamic/static inventory split, and the Ansible-to-Flux handoff model.

**[GitOps Model](gitops-model.md)** — The Flux feature catalog, the three-tier overlay ownership model (template base → user-global → per-cluster), and day-2 operations.
