---
title: Guides
description: Task-oriented how-to guides covering the full Talos Kubernetes cluster lifecycle, from initial setup to day-2 operations.
type: index
audience: [user, operator]
tags: [guides, talos, kubernetes]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - ../index.md
  - ../concepts/index.md
---

# Guides

Task-oriented how-to guides for every stage of the cluster lifecycle. Each guide answers "how do I do X?" with numbered steps and expected outcomes.

## Pages

| Guide | Description |
|---|---|
| [Getting Started](getting-started.md) | Prerequisites and the full bootstrap sequence from zero to a running cluster |
| [Configure Your Cluster](configure-your-cluster.md) | Authoring `cluster.yaml`, `nodes.yaml`, and Talos config overlays |
| [Manage Nodes and Roles](manage-nodes-and-roles.md) | Adding, removing, and reconfiguring nodes; assigning roles |
| [Add a Provider](add-a-provider.md) | Porting the template to a new cloud provider or bare-metal environment |
| [Upgrades](upgrades.md) | Upgrading Talos, Kubernetes, and Flux |
| [Storage](storage.md) | Configuring local and distributed storage (OpenEBS and others) |
| [Ingress and TLS](ingress-and-tls.md) | Setting up ingress controllers and cert-manager for TLS |
| [GitOps with Flux](gitops-with-flux.md) | Understanding and operating the Flux GitOps workflow |
| [Disaster Recovery](disaster-recovery.md) | Backup, restore, and cluster recovery procedures |


