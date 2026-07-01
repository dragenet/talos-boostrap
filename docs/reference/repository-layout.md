---
title: Repository Layout
description: Annotated directory tree explaining the purpose of every top-level folder in the Talos bootstrap template repo.
type: reference
audience: [user, contributor]
tags: [repo, layout]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - cluster-yaml-schema.md
  - playbooks.md
---

# Repository Layout

This page is a reference map of the repository structure. Each top-level directory has a single, well-defined responsibility; the boundaries are enforced by the one-writer-per-file rule (ADR-011).

## Top-level directories

| Directory | Owner | Purpose |
|---|---|---|
| `config/` | User + Template | Cluster configuration inputs (defaults, user overrides, schemas, Talos patches) |
| `flux/` | User | FluxCD manifests — apps, infrastructure, and cluster entry-points |
| `infra/` | Template | Ansible playbooks, roles, and inventories that bootstrap the cluster |
| `dev-docs/` | Template (legacy) | Original design ADRs and notes; being superseded by `docs/` |
| `docs/` | Template | User-facing documentation vault (this site) |
| `.ai/` | Template | AI-assistant context files and prompt helpers |
| `.github/` | Template | GitHub Actions workflows and CI configuration |

---

## `config/` subtree

The config compiler (`render-config.yml`) reads from `config/` and emits generated artifacts. Users edit only `config/clusters/<cluster>/`; template updates land in `config/defaults/`.

```
config/
├── defaults/
│   └── cluster.yaml          # Template-owned defaults — do NOT edit
├── schemas/
│   ├── cluster-input.schema.json   # JSON Schema for cluster.yaml
│   └── nodes.schema.json           # JSON Schema for nodes.yaml
├── clusters/
│   └── <cluster-name>/       # One directory per cluster (user-owned)
│       ├── cluster.yaml      # Cluster-specific overrides (deep-merged over defaults)
│       ├── nodes.yaml        # Optional static node list
│       └── talos/            # Optional per-cluster Talos patch overlays
└── talos/
    ├── base/                 # Shared Talos machine-config base patches (template-owned)
    └── components/           # Talos component patches (template-owned)
```

| Path | Purpose |
|---|---|
| `config/defaults/cluster.yaml` | Template-owned baseline; every field has a documented default |
| `config/schemas/` | JSON Schema files that validate cluster and node inputs |
| `config/clusters/<cluster>/cluster.yaml` | User-owned cluster settings; merged over defaults at render time |
| `config/clusters/<cluster>/nodes.yaml` | Optional static node declarations for the manual provider |
| `config/clusters/<cluster>/talos/` | Optional per-cluster Talos config patch overlays |
| `config/talos/base/` | Shared Talos machine-config patches applied to all clusters |
| `config/talos/components/` | Feature-gated Talos component patches |

---

## `infra/` subtree

Ansible lives here. The playbooks are the only way to bootstrap or upgrade the cluster; roles are the internal units of work called by playbooks.

```
infra/
└── ansible/
    ├── ansible.cfg
    ├── requirements.yml
    ├── playbooks/
    │   ├── render-config.yml
    │   ├── bootstrap-talos.yml
    │   ├── bootstrap-flux.yml
    │   ├── upgrade-talos.yml
    │   └── providers/
    │       └── hcloud/
    │           ├── image-talos.yml
    │           └── provision-infra.yml
    ├── roles/
    │   ├── config_render/      # Reads cluster.yaml; emits generated group_vars and patches
    │   ├── talos_schematic/    # Resolves Talos extensions → schematic ID via factory.talos.dev
    │   ├── talos_config/       # Generates and applies Talos machine configs; bootstraps etcd
    │   ├── talos_upgrade/      # Upgrades Talos OS one node at a time with health checks
    │   ├── flux_bootstrap/     # Installs pre-Flux components (Cilium, CCM) and bootstraps Flux
    │   └── providers/
    │       └── hcloud/         # Hetzner Cloud provider roles (server provisioning)
    └── inventories/
        ├── common/             # Shared group_vars and localhost host definition
        └── providers/
            ├── hcloud/         # Hetzner Cloud: dynamic inventory plugin config + group_vars
            └── manual/         # Manual/bare-metal: static hosts.yml
```

| Path | Purpose |
|---|---|
| `playbooks/` | Operator-facing entrypoints; see [Playbooks reference](playbooks.md) |
| `roles/config_render/` | Config compiler role; single writer for generated group_vars |
| `roles/talos_schematic/` | Resolves extension list → deterministic schematic ID |
| `roles/talos_config/` | Applies machine configs and bootstraps etcd |
| `roles/talos_upgrade/` | Sequenced, health-checked OS upgrade per node |
| `roles/flux_bootstrap/` | Pre-Flux add-ons (Cilium, hcloud-ccm) and Flux bootstrap |
| `inventories/common/` | Variables shared across all providers |
| `inventories/providers/hcloud/` | Hetzner Cloud dynamic inventory and provider group_vars |
| `inventories/providers/manual/` | Static inventory for bare-metal or manually provisioned nodes |

---

## `flux/` subtree

GitOps manifests reconciled by FluxCD after bootstrap. Organised as a standard multi-cluster Flux repository.

```
flux/
├── apps/          # Application HelmReleases and Kustomizations (user workloads)
├── infrastructure/# Cluster infrastructure (cert-manager, ingress, storage, etc.)
└── clusters/      # Per-cluster Flux entry-points; references apps/ and infrastructure/
```

| Path | Purpose |
|---|---|
| `flux/clusters/<cluster>/` | Flux entry-point Kustomization for a specific cluster |
| `flux/infrastructure/` | Shared infrastructure components deployed by Flux |
| `flux/apps/` | User application definitions deployed by Flux |

---

## Supporting directories

| Directory | Purpose |
|---|---|
| `.ai/` | Context files for AI-assisted development (not for human consumption) |
| `.github/` | GitHub Actions CI workflows, including drift-detection for generated files |
