# ADR-010: Ansible-to-Flux provider handoff — pre-Flux layer and provider overlays

**Status:** Accepted

## Context

Bootstrap has two distinct phases with different tools:

1. **Ansible phase** — provisions infrastructure, applies Talos machine configs, bootstraps etcd, installs pre-Flux components (CNI, provider CCM if enabled, later CSI). Provider-aware, runs once per cluster lifecycle.
2. **Flux phase** — reconciles all remaining cluster components from git. Provider-agnostic in principle, but some components (Hetzner CSI, cloud-specific StorageClasses) are provider-specific.

The handoff between these phases must be explicit: Flux must not attempt to reconcile provider-specific components on a cluster whose provider doesn't support them (e.g. Hetzner CSI on bare-metal). Additionally, core bootstrap components that are required before Flux can schedule pods must be installed by Ansible first and then adopted by Flux.

## Decision

Core bootstrap configuration lives outside the optional `features` catalog:

- `cni.name` selects the CNI (see ADR-002).
- `cloud_controller_manager.enabled` controls CCM enablement (see ADR-005).
- `features` (ADR-012) remains for optional platform / app features such as OpenEBS, Envoy Gateway, Traefik, external-dns, cert-manager, and ingress.

The pre-Flux layer is driven by these core config keys plus provider:

- Ansible's `bootstrap-flux.yml` installs:
  - Cilium if `cni.name: cilium`;
  - the provider's CCM if `cloud_controller_manager.enabled` resolves to `true`;
  - (future) provider CSI when applicable.
- Each pre-Flux component is installed via Helm into its own namespace (`cilium-system`, `hcloud-ccm-system`, etc.) using release names and values that match the Flux-managed HelmRelease. This lets Flux adopt them without conflict after `flux bootstrap`.

After `flux bootstrap` runs, Flux picks up the provider overlay from the cluster entrypoint and reconciles the same components. The Flux infrastructure tree uses the catalog + render-generated base + user overlay layout (see ADR-011/012):

```
flux/infrastructure/controllers/
  _components/     # TEMPLATE-OWNED catalog of component bases (incl. providers/<name>/ for provider-gated)
  base/            # RENDER-OWNED: the enabled components for this cluster (from core config + provider + features)
  patches/         # USER-OWNED: kustomize strategic-merge patches
  kustomization.yaml  # USER-OWNED overlay: resources:[base, ...]; patches:[patches/*]
```

The cluster entrypoint (`flux/clusters/<cluster>/infrastructure.yaml`) points at the user overlay. Provider selection flows from `cluster.yaml` `provider:` → the render includes the provider's components in `base/`. Per-component customization is a patch (tier 2); replacing a component is disable-and-deploy-your-own (tier 3) — see ADR-011.

### What goes where

| Component | Ansible pre-Flux | Flux (rendered into `base/`) | Reason |
|---|---|---|---|
| CNI: Cilium (if `cni.name: cilium`) | ✅ install | ✅ adopt | CNI must exist before pods schedule. `flannel` needs no pre-Flux install. |
| Provider CCM (if enabled) | ✅ install | ✅ adopt | Must clear `uninitialized` taint before Flux controllers schedule. |
| cert-manager | — | ✅ | Optional feature. |
| Envoy Gateway / Traefik | — | ✅ | Optional feature (ingress). |
| OpenEBS LocalLV LVM | — | ✅ | Optional feature (+ Talos volume, ADR-013). |
| Hetzner CSI / `hcloud-volumes` SC | — | ✅ | Provider-gated feature (`_components/providers/hcloud/`). |

## Consequences

- Adding a new provider follows the single `providers/<name>/` rule (ADR-007): a catalog dir `_components/providers/<name>/`, an inventory, config, and the render including it in `base/`.
- The pre-Flux component list is rendered from `cni.name`, `cloud_controller_manager.enabled`, and `provider.name`; `manual` disables CCM automatically and omits cloud-specific CSI.
- Components that cannot tolerate the `uninitialized` taint must stay out of the Ansible pre-Flux layer — they belong in Flux where the taint is already cleared.
- `cluster.yaml` `provider:` is the single source for provider selection; the render derives both the Ansible inventory expectation and the Flux `base/` contents from it, so they cannot diverge.
- Pre-Flux components run in dedicated `<component>-system` namespaces rather than `kube-system`, making ownership and RBAC clearer.

> **Implementation status (2026-06-26):** Implemented. `flux bootstrap`, the Ansible pre-Flux install of Cilium / CCM, and the Flux adoption step are built. The `flux_bootstrap` role sequences preflight → namespaces → Cilium → hcloud CCM → Flux bootstrap. The render-generated `flux/infrastructure/controllers/generated/selected/kustomization.yaml` now includes enabled core components such as `../../_components/cilium` and `../../_components/providers/hcloud/ccm`, following the four-tier overlay model (ADR-017).
