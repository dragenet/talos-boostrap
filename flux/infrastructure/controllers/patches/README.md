# User-owned patches

**Owner:** Cluster operators. Edit freely — render will never overwrite this directory.

This directory holds kustomize strategic-merge patches applied on top of the
render-generated `base/` kustomization. Patches let you customize component
values (e.g. HelmRelease `.spec.values`) without replacing the entire component.

## Rules

- Every patch **must target an enabled component**. A patch referencing a
  disabled or missing component will break the kustomize build (the target
  resource won't exist). See ADR-012 constraints.
- Patches are applied by the user-owned overlay at
  `flux/infrastructure/controllers/kustomization.yaml`. When adding a patch,
  add its path to the `patches:` list in that file.
- For component replacement (not just tweaking values), disable the feature in
  `config/overrides/cluster.yaml` and deploy your own manifest via Flux (tier 3
  in ADR-011's customization ladder).

## Current state

Empty — no patches yet. When feature work lands and components are enabled in
`base/`, add patches here as needed.

## Example

```yaml
# patches/cilium-values.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cilium
  namespace: kube-system
spec:
  values:
    ipam:
      mode: kubernetes
```

Then in `flux/infrastructure/controllers/kustomization.yaml`:

```yaml
patches:
  - path: patches/cilium-values.yaml
```
