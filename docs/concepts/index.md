---
title: Concepts
description: Explanations of how the Talos Kubernetes bootstrap template works — architecture, config compilation, GitOps, and provider portability.
type: index
audience: [user, operator, ai]
tags: [concepts, talos, kubernetes, architecture]
status: draft
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

_Pages listed above are coming in a later step._
