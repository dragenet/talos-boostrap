# ADR-012: Feature catalog and taxonomy

**Status:** Accepted

## Context

Cluster components (cert-manager, ingress, OpenEBS, CSI, …) are not a fixed set — each cluster enables a subset, some by user choice, some gated by provider capability. Enabling/disabling must be expressible without forking template files, and without runtime mutation of what GitOps manages.

## Decision

Components form a **catalog** of template-owned bases under `flux/infrastructure/controllers/_components/`. A cluster's selection is rendered into the `base/` kustomization from `cluster.yaml` `features:` (see ADR-011). **Enabling a feature is structural** (which component the kustomization references) — never a runtime value, because kustomize has no conditionals.

### Taxonomy

| Type | Examples | Toggle | Ansible? |
|---|---|---|---|
| **pure-Flux** | cert-manager, ingress (envoy-gateway / traefik) | `features:` → render writes `base/` | render to enable; patch is git-only |
| **Talos-coupled** | `cni`, openebs | `features:` → render writes `base/` **and** the Talos patch | yes (Talos half) |
| **provider-gated** | Hetzner CSI, CCM | provider overlay (`resources:`) | provider choice; CCM is pre-Flux |
| **secret-backed** | cert-manager (Cloudflare), Tailscale | feature + SOPS secret in git | only the generic `sops-age` inject |

### Customization & eject
- **Tweak:** kustomize strategic-merge patch in the user overlay `patches/` subdir (works on HelmRelease `.spec.values` and raw manifests alike).
- **Replace:** disable the feature in `cluster.yaml`, deploy your own manifest via Flux. Ejecting a **Talos-coupled** feature only ejects its Flux half — the Talos substrate (disk/network) still comes from `cluster.yaml`.

### Constraints (enforced by render validation)
- ingress is **pick-one** (envoy-gateway XOR traefik XOR none);
- a feature needing certs requires `cert-manager: true`;
- provider-gated features rejected on `manual`;
- every user patch must target an **enabled** component (else a disabled feature orphans its patch and breaks the kustomize build).

## Consequences

- Adding a feature to the template = a new `_components/<feature>/` base + a schema entry + (if Talos-coupled) render logic.
- The `base/` kustomization is render-owned; users never hand-edit it — they patch (tier 2) or eject (tier 3) per ADR-011's ladder.
