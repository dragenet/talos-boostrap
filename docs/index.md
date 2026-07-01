---
title: Documentation Home
description: Hub for the Talos Kubernetes bootstrap template docs — guides, concepts, reference, ADRs, and examples.
type: index
audience: [user, contributor, operator, ai]
tags: [talos, kubernetes, flux, ansible]
status: stable
created: 2026-07-01
updated: 2026-07-01
---

# Talos Kubernetes Bootstrap Template — Documentation

This repo is a reusable template for deploying production-grade Talos Linux Kubernetes clusters using Ansible for provisioning, Flux for GitOps, and a config compiler that renders per-cluster machine configs from a shared defaults layer. It is designed to be forked and adapted to any cloud provider or bare-metal environment.

Use the sections below to navigate the vault. Each section index lists the pages it contains.

## Sections

| Section | Description |
|---|---|
| [Guides](guides/index.md) | Task-oriented how-to guides covering the full cluster lifecycle |
| [Concepts](concepts/index.md) | Explanations of how the template works and why it is designed this way |
| [Reference](reference/index.md) | Schemas, playbook reference, inventory contract, and glossary |
| [ADRs](adr/index.md) | Architecture decision records for generic template decisions |
| [Contributing](contributing/index.md) | Dev setup, docs style guide, writing ADRs, and working with AI agents |
| [Example: kube1 on Hetzner](examples/kube1-hetzner/index.md) | Concrete worked example — a real cluster deployed on Hetzner Cloud |

## Quick orientation

- **New here?** Start with [Getting Started](guides/getting-started.md) then read the [Architecture Overview](concepts/architecture-overview.md).
- **Configuring a cluster?** See [Configure Your Cluster](guides/configure-your-cluster.md) and the [cluster.yaml Schema](reference/cluster-yaml-schema.md).
- **Adding a provider?** Read [Add a Provider](guides/add-a-provider.md) and [Provider Portability](concepts/provider-portability.md).
- **Contributing docs?** Read the [Docs Style Guide](contributing/docs-style-guide.md) first.
- **AI agent?** Every page carries frontmatter with `description`, `type`, `tags`, and `related` paths for retrieval. The style guide is at [contributing/docs-style-guide.md](contributing/docs-style-guide.md).
