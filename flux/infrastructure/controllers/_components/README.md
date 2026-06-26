# Flux controller catalog (`_components/`)

**Owner:** Template maintainers. Do not edit without updating the render compiler.

This directory is a **sparse catalog** of controller component bases. Each
subdirectory is an ordinary kustomize base that can be referenced from
`../generated/selected/kustomization.yaml` when the render compiler enables it for a cluster.

## Catalog contract

- Each entry is a self-contained kustomize base with its own `kustomization.yaml`.
- Entries that introduce `HelmRepository`, `HelmRelease`, or other Kubernetes
  resources **must be ordinary kustomize bases** (`kind: Kustomization` with
  `resources:`), not official Kustomize `kind: Component` overlays.
- Official Kustomize `kind: Component` entries are transform-only — they cannot
  contain `resources:` or nested `components:`. Since controller catalog entries
  need to introduce resources (HelmReleases, CRDs, RBAC, etc.), they are
  ordinary bases by default.
- A future entry that is genuinely transform-only (e.g. a label transformer or
  name prefix applied across other bases) may use `kind: Component` and be
  listed under `components:` in `../generated/selected/kustomization.yaml`. That is not the
  default for controllers.

## Directory structure

```
_components/
  <feature>/              # e.g. cilium, cert-manager, openebs
    kustomization.yaml    # ordinary base: resources, helm repos/releases
  providers/
    <provider>/           # e.g. hcloud, manual
      <feature>/          # provider-gated components (e.g. hetzner-csi)
        kustomization.yaml
```

Provider-gated components live under `providers/<name>/` and are only included
in `generated/selected/` when `cluster.yaml` `provider.name` matches. See ADR-010 and ADR-012.

## Current state

Implemented catalog entries (each a full Namespace + HelmRepository +
HelmRelease base, chart versions pinned, ready for Flux adoption):

- `cilium/` — Cilium CNI (kube-proxy replacement, VXLAN tunnel; chart 1.19.5).
- `providers/hcloud/ccm/` — Hetzner Cloud Controller Manager (chart 1.33.0).

The render compiler references the enabled entries from
`../generated/selected/kustomization.yaml` (render-owned). Future entries
(hcloud CSI, cert-manager, OpenEBS, ingress) follow the same pattern and are
added by flipping the matching `features.*` flag in the selected-cluster
config and re-rendering.

## Adding a new catalog entry

1. Create `_components/<feature>/kustomization.yaml` as an ordinary base.
2. Add the feature toggle to `config/defaults/cluster.yaml` schema.
3. Update the render compiler to include the entry in `../generated/selected/` when enabled.
4. If Talos-coupled (e.g. CNI, OpenEBS), also update Talos patch rendering.
