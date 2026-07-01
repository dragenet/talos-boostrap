---
title: GitOps Model
description: How Flux watches the flux/ tree, the feature catalog model, overlay ownership, and day-2 operations.
type: concept
audience: [user, contributor]
tags: [flux, gitops, kubernetes]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - architecture-overview.md
  - config-compiler.md
  - ../adr/0006-fluxcd-gitops.md
  - ../adr/0012-feature-catalog.md
  - ../adr/0017-shared-overlay-ownership.md
---

# GitOps Model

After the initial bootstrap, all cluster state is managed declaratively from git using FluxCD. Flux watches the `flux/` tree in this repository and reconciles `Kustomization` and `HelmRelease` objects automatically. Committing a change to `flux/` is the primary day-2 operation.

## How Flux Watches the Repo

The cluster entrypoint is `flux/clusters/<cluster>/`. It references two top-level kustomizations with explicit `dependsOn` ordering:

```
infrastructure/controllers → infrastructure/configs → apps
```

All Flux `Kustomization` resources set `prune: true` — resources removed from git are removed from the cluster. Infrastructure layers use `wait: true` so dependent layers do not reconcile until their dependencies are healthy. Helm chart and image versions are pinned; no floating `:latest` tags.

## The Feature Catalog

Cluster components — cert-manager, ingress controllers, OpenEBS, cloud CSI drivers — form a **catalog** of template-owned bases under `flux/infrastructure/controllers/_components/`. Each entry is a complete, tested component definition.

A cluster's active component set is rendered into `flux/infrastructure/controllers/generated/selected/` by the config compiler from `cluster.yaml` `features:` and core config keys. **Enabling a feature is structural**: the render adds a reference to the component's base in the generated kustomization. Kustomize has no conditionals, so presence in the kustomization is the only mechanism.

### Feature Taxonomy

| Type | Examples | How to enable | Ansible required? |
|---|---|---|---|
| **Core bootstrap** | CNI (`cni.name`), CCM (`cloud_controller_manager.enabled`) | Top-level config keys | Yes — pre-Flux install, then Flux adopts |
| **Pure-Flux** | cert-manager, Envoy Gateway, Traefik | `features:` in `cluster.yaml` | Render only |
| **Talos-coupled** | OpenEBS LVM | `features:` + Talos volume config | Yes (Talos half requires render + bootstrap) |
| **Provider-gated** | Cloud CSI drivers | Provider overlay | Provider selection |
| **Secret-backed** | cert-manager (Cloudflare), Tailscale | Feature + SOPS secret in git | Only `sops-age` key inject |

### Constraints

The render enforces several feature constraints:

- Ingress is pick-one: `envoy-gateway` XOR `traefik` XOR none.
- Features that need TLS certificates require `cert-manager: true`.
- Provider-gated features are rejected if `provider.name: manual`.
- Every user patch must target an enabled component; a patch targeting a disabled feature breaks the kustomize build.

## Overlay Ownership (Three-Tier Model)

ADR-017 establishes a three-tier ownership model for both Talos and Flux inputs:

1. **Template/shared base** — reusable, cluster-agnostic defaults owned by the template (`_components/`, `base/`).
2. **User-global overlay** — preferences that apply to every cluster in this repo (`overlays/user/` or `overlays/common/`).
3. **Per-cluster overlay** — final overrides for a specific cluster (`overlays/clusters/<cluster>/`).

Later layers override earlier layers. The per-cluster overlay is the final user-owned layer.

The illustrative Flux tree layout:

```
flux/
  infrastructure/
    controllers/
      components/              # template-owned component catalog
      base/                    # render-selected enabled components
      overlays/
        user/                  # repo-wide patches applied to all clusters
        clusters/
          <cluster>/           # per-cluster patches
    configs/
      base/
      overlays/
        user/
        clusters/
          <cluster>/
  apps/
    base/
    overlays/
      user/
      clusters/
        <cluster>/
  clusters/
    <cluster>/
      flux-system/             # owned by flux bootstrap — do not edit
      infrastructure.yaml
      apps.yaml
```

The `flux/clusters/<cluster>/flux-system/` directory is owned by `flux bootstrap`. Do not hand-edit it.

## Customizing Features

Following the three-tier model, there are two ways to customize a feature without replacing it entirely:

- **Tweak (tier 2)** — add a Kustomize strategic-merge or JSON patch in `overlays/user/` (repo-wide) or `overlays/clusters/<cluster>/patches/` (per-cluster). This works on `HelmRelease .spec.values` and raw manifests. This is a pure git change; Flux reconciles it automatically.
- **Replace (tier 3)** — disable the feature in `cluster.yaml` (the render removes it from `generated/selected/`), then deploy your own version via Flux. For Talos-coupled features (e.g. OpenEBS), ejecting the Flux half does not remove the Talos substrate; the disk/volume config in `cluster.yaml` still applies.

## Day-2 Operations

| Operation | How |
|---|---|
| Add an app | Add a `HelmRelease` or `Kustomization` under `flux/apps/`; commit and push |
| Enable a feature | Set it in `cluster.yaml`, run `render-config.yml`, commit generated output, push |
| Patch a feature | Add a patch in `overlays/clusters/<cluster>/patches/`; commit and push |
| Disable a feature | Unset it in `cluster.yaml`, run render, commit and push |
| Rotate a SOPS secret | Re-encrypt the secret file; commit and push |
| Upgrade a Helm chart | Update the `version:` in the HelmRelease; commit and push |
| Talos machine-config change | Update `cluster.yaml` or `nodes.yaml`, run render, run `bootstrap-talos.yml` |

## Pre-Flux Components

Cilium (CNI) and the provider CCM must be running before Flux can schedule its own controllers. `bootstrap-flux.yml` installs them via Helm with release names and values that match the Flux-managed HelmRelease. After `flux bootstrap` runs, Flux adopts these pre-installed releases without conflict; it does not reinstall them.

## Further Reading

- [Architecture Overview](architecture-overview.md) — overall system map
- [Config Compiler](config-compiler.md) — how features are selected during render
- [ADR-006: FluxCD for GitOps](../adr/0006-fluxcd-gitops.md)
- [ADR-012: Feature Catalog](../adr/0012-feature-catalog.md)
- [ADR-017: Shared Overlay Ownership](../adr/0017-shared-overlay-ownership.md)
