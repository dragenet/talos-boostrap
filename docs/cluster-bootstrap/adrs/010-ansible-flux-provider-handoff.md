# ADR-010: Ansible-to-Flux provider handoff — pre-Flux layer and provider overlays

**Status:** Accepted

## Context

Bootstrap has two distinct phases with different tools:

1. **Ansible phase** — provisions infrastructure, applies Talos machine configs, bootstraps etcd, installs pre-Flux components (Cilium, provider CCM if needed). Provider-aware, runs once per cluster lifecycle.
2. **Flux phase** — reconciles all remaining cluster components from git. Provider-agnostic in principle, but some components (Hetzner CSI, cloud-specific StorageClasses) are provider-specific.

The handoff between these phases must be explicit: Flux must not attempt to reconcile provider-specific components on a cluster whose provider doesn't support them (e.g. Hetzner CSI on bare-metal).

## Decision

The Flux infrastructure tree uses a **catalog + render-generated base + user overlay** layout (see ADR-011/012 for ownership and the customization ladder):

```
flux/infrastructure/controllers/
  _components/     # TEMPLATE-OWNED catalog of component bases (incl. providers/<name>/ for provider-gated)
  base/            # RENDER-OWNED: the enabled components for this cluster (from cluster.yaml features + provider)
  patches/         # USER-OWNED: kustomize strategic-merge patches
  kustomization.yaml  # USER-OWNED overlay: resources:[base, ...]; patches:[patches/*]
```

The cluster entrypoint (`flux/clusters/<cluster>/infrastructure.yaml`) points at the user overlay. Provider selection flows from `cluster.yaml` `provider:` → the render includes the provider's components in `base/`. Per-component customization is a patch (tier 2); replacing a component is disable-and-deploy-your-own (tier 3) — see ADR-011.

The Ansible `bootstrap-flux.yml` playbook installs the pre-Flux layer from a provider-keyed variable (e.g. `bootstrap_pre_flux_components: [cilium, hcloud-ccm]`). After `flux bootstrap` runs, Flux picks up the provider overlay from the cluster entrypoint and reconciles the provider-specific controllers.

### What goes where

| Component | Ansible pre-Flux | Flux (rendered into `base/`) |
|---|---|---|
| Cilium (if `cni: cilium`) | ✅ (CNI must exist before pods) | adopted by Flux post-bootstrap (Option B, ADR-005-pattern) |
| hcloud CCM (cloud only) | ✅ (must clear taint before Flux) | adopted by Flux |
| cert-manager | — | ✅ |
| Envoy Gateway | — | ✅ |
| OpenEBS LocalLV LVM | — | ✅ (+ Talos volume, ADR-013) |
| Hetzner CSI / `hcloud-volumes` SC | — | ✅ (provider-gated: `_components/providers/hcloud`) |

## Consequences

- Adding a new provider follows the single `providers/<name>/` rule (ADR-007): a catalog dir `_components/providers/<name>/`, an inventory, config, and the render including it in `base/`.
- The pre-Flux component list (`bootstrap_pre_flux_components`) is rendered from `cluster.yaml` — `manual` sets `[cilium]` (or none, if `flannel`); hcloud adds `hcloud-ccm`.
- Components that cannot tolerate the `uninitialized` taint must stay out of the Ansible pre-Flux layer — they belong in Flux where the taint is already cleared.
- `cluster.yaml` `provider:` is the single source for provider selection; the render derives both the Ansible inventory expectation and the Flux `base/` contents from it, so they cannot diverge.

> Note: in VXLAN mode the CCM route controller is **not** used (ADR-002, kube1 ADR-006). CCM is pre-Flux purely to clear the taint / set providerID; native-routing route programming is out of scope here.
