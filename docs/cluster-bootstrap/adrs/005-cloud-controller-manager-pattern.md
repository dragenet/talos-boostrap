# ADR-005: Cloud controller manager — tri-state enablement and provider resolution

**Status:** Accepted

## Context

When kubelet and kube-controller-manager run with `--cloud-provider=external`, Kubernetes taints every node with `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule`. Only a CCM can clear this taint, and until it does no workloads schedule — including Flux's own controllers. CCM must therefore run **before** `flux bootstrap`, installed directly by Ansible, and then be adopted by Flux as steady state.

The bootstrap template supports both cloud providers (e.g. `hcloud`) and a metal/null provider (`manual`). CCM availability is provider-specific, but the enablement decision must be expressed in a provider-agnostic way.

## Decision

`cloud_controller_manager.enabled` is a core cluster config key (not a `features` entry) with three values:

| Value | Behaviour |
|---|---|
| `auto` (default) | Render resolves the provider. If a known CCM exists for `provider.name`, enable it; otherwise disable it. |
| `true` | Enable the external cloud provider and install the provider's CCM. Render / validation fails if the provider has no supported CCM. |
| `false` | Disable the external cloud provider; no CCM is installed. |

Rules:

- `cloud_controller_manager.enabled` is provider-agnostic; provider-specific CCM selection is handled by the render's `provider.name` mapping.
- If CCM is enabled, it is installed pre-Flux by Ansible and then adopted by Flux (ADR-010), using the namespace `<provider>-ccm-system` (e.g. `hcloud-ccm-system`) and matching Helm release name and values.
- If CCM is disabled or the provider has no supported CCM, `externalCloudProvider.enabled` must remain `false` so kubelet / kube-controller-manager do not run with `--cloud-provider=external`.
- `features` is reserved for optional platform / app features (OpenEBS, Envoy Gateway, Traefik, external-dns, etc.) and must not be used for CCM enablement.

### hcloud CCM configuration (reference implementation)

When `provider.name: hcloud` and CCM is enabled, deploy the hcloud CCM via Helm with `networking.enabled: false`. Active roles:

- **Kubernetes cloud controller** — node lifecycle, `providerID = hcloud://<id>`, `InternalIP` from private network, `topology.kubernetes.io/zone` / `region` labels, clears uninitialized taint.
- **CSI prerequisite** — Hetzner CSI requires `providerID` on each node to attach Cloud Volumes.

Disabled roles:

- **Route controller** (`networking.enabled: false`) — not needed; CNI uses VXLAN (ADR-002).
- **Load-balancer controller** — not needed; ingress uses host-network Envoy Gateway (ADR-004).

## Consequences

- CCM is reconciled by Flux after bootstrap; it is not a one-off Ansible install.
- Adding a new cloud provider to the bootstrap template requires registering its CCM in the render's auto-resolution table and adding its catalog base under `flux/infrastructure/controllers/_components/providers/<name>/`.
- The pre-Flux Ansible component list is rendered from `cni.name` and `cloud_controller_manager.enabled`; `manual` disables CCM automatically.
- Components that cannot tolerate the uninitialized taint must not be placed in the Ansible pre-Flux layer — they belong in Flux, where the taint is already cleared.
- Enabling the external cloud provider without a supported CCM leaves nodes permanently tainted; `true` therefore requires validation.

> **Implementation status (2026-06-23):** Implemented. The `cloud_controller_manager.enabled` tri-state schema migration is complete (`features.hcloudCcm` is no longer used). The `flux_bootstrap` role installs the hcloud CCM pre-Flux when the tri-state resolves to enabled, the matching `HelmRelease` under `flux/infrastructure/controllers/_components/providers/hcloud/ccm/` adopts it, and the render emits the `cluster.externalCloudProvider` Talos patch.
