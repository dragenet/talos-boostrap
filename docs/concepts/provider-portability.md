---
title: Provider Portability
description: How the template isolates provider-specific code behind an inventory contract so playbooks and Talos roles stay provider-agnostic.
type: concept
audience: [user, contributor]
tags: [ansible, providers, portability]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - architecture-overview.md
  - ../adr/0007-ansible-portable-bootstrap.md
  - ../adr/0010-ansible-flux-provider-handoff.md
  - ../adr/0014-node-discovery-dynamic-inventory.md
---

# Provider Portability

The template is designed to deploy a Talos Kubernetes cluster on any supported infrastructure provider without forking playbooks or roles. Provider-specific code lives in exactly one place per tree; everything else is provider-agnostic by construction.

## The Core Rule

One uniform rule applies across every tree in the repo:

> **Provider-agnostic at the top. Provider-specific under `providers/<name>/`.**

This applies to Ansible inventories, playbooks, roles, and Flux infrastructure components. Adding a new provider means adding a new `providers/<name>/` directory in each relevant tree — nothing outside those directories changes.

## The Inventory Contract

Provider-agnostic Ansible roles never reference provider APIs directly. Instead, they read standard hostvars that every provider must supply for each node:

| Hostvar | Description |
|---|---|
| `node_public_ip` | Reachable IP for Talos API calls |
| `node_private_ip` | Private IP for VIP/etcd addressing |
| `node_role` | Node role: `controlplane`, `hybrid`, or `worker` |

Nodes are also placed into Ansible groups (`controlplane`, `hybrid`, `worker`) derived from their role. Roles such as `talos_config` and `talos_upgrade` read `hostvars[node].node_public_ip` and `.node_private_ip` exclusively — they never name a provider.

### Dynamic vs. Static Inventory

- **Cloud providers** (e.g. hcloud): the provider's dynamic inventory plugin discovers live servers after provisioning and populates the hostvars above. A label selector scopes discovery to the current cluster.
- **Manual (bare-metal)**: a checked-in `hosts.yml` declares the same hostvars statically.

Both mechanisms produce identically-shaped hostvars; provider-agnostic roles see no difference.

The previous design wrote a `node_ips.yml` file as cross-play IPC. This was provider-shaped and non-idempotent across separate `ansible-playbook` invocations. ADR-014 removed it: node addresses and roles now flow only through standard Ansible inventory.

## The Handoff Model

Bootstrap has two phases with a clean handoff between them:

### Phase 1 — Ansible (provider-aware)

Runs once per cluster lifecycle:

1. **Provision** — provider playbook creates VMs/nodes, assigns IPs, attaches them to the private network. Lives under `infra/ansible/playbooks/providers/<provider>/`.
2. **Render** — `render-config.yml` generates Talos patches and Flux artifacts from `config/`.
3. **Bootstrap Talos** — `bootstrap-talos.yml` applies machine configs and bootstraps etcd. Provider-agnostic.
4. **Bootstrap Flux** — `bootstrap-flux.yml` installs pre-Flux components (CNI, provider CCM if enabled), then runs `flux bootstrap`. CNI and CCM are installed with Helm using release names and values that match the Flux-managed HelmRelease, so Flux can adopt them without conflict.

### Phase 2 — Flux (provider-agnostic in principle)

After `flux bootstrap`, Flux reconciles ongoing cluster state from git. Provider-specific Flux components (cloud CSI drivers, cloud-specific StorageClasses) are gated behind provider overlays in the feature catalog and are never reconciled on a provider that does not support them.

## Provider Selection

`cluster.yaml` `provider:` is the single source for provider selection. The render derives both:

- which Ansible inventory to use (`inventories/common` + `inventories/providers/<name>`); and
- which Flux components are included in `flux/infrastructure/controllers/generated/selected/`.

The two cannot diverge because they share the same input.

## The `manual` Provider

`manual` (bare-metal) is modelled as the "null provider":

- It sits under `providers/` for structural symmetry.
- It has the smallest surface: no CCM, no cloud CSI, no dynamic inventory.
- It uses a static `hosts.yml` with the same hostvar contract as dynamic providers.

## Adding a New Provider

At a high level, adding a provider requires:

1. A `providers/<name>/` directory in `infra/ansible/inventories/` with a matching inventory.
2. A `providers/<name>/` directory in `infra/ansible/playbooks/` for provisioning playbooks.
3. A `providers/<name>/` directory in `flux/infrastructure/controllers/_components/` for provider-gated Flux components (e.g. CSI driver).
4. Hostvars that satisfy the inventory contract (`node_public_ip`, `node_private_ip`, `node_role`).

No changes to provider-agnostic playbooks or roles are required. See `guides/add-a-provider.md` for the step-by-step procedure.

## Further Reading

- [Architecture Overview](architecture-overview.md) — overall system map
- [ADR-007: Ansible Portable Bootstrap](../adr/0007-ansible-portable-bootstrap.md)
- [ADR-010: Ansible-to-Flux Provider Handoff](../adr/0010-ansible-flux-provider-handoff.md)
- [ADR-014: Node Discovery via Inventory](../adr/0014-node-discovery-dynamic-inventory.md)
