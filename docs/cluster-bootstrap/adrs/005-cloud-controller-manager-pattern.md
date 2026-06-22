# ADR-005: Cloud controller manager — enabled for cloud providers, disabled for metal

**Status:** Superseded by ADR-010

> **Supersession note:** ADR-010 refactors the bootstrap architecture so that the provider CCM is still installed pre-Flux by Ansible (to clear the `uninitialized` taint in time for Flux), but is then **adopted by Flux** and reconciled from git as steady state. The "CCM runs outside Flux, not reconciled" consequence below no longer applies to kube1's refactored architecture; see ADR-010 §What goes where for the current ownership table. The provider-type distinction (cloud installs CCM, metal does not) still holds.

## Context

When kubelet starts with `--cloud-provider=external`, Kubernetes taints every node with `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule`. Only a CCM can clear this taint, and until it does no workloads schedule — including Flux's own controllers. CCM must therefore run **before** `flux bootstrap`, installed directly by Ansible.

The bootstrap template supports two provider types:

- **Cloud providers** (e.g. `hcloud`) — have a CCM that integrates Kubernetes with the cloud API: node lifecycle (removes the Node object when the VM is deleted), `providerID`, node addresses, zone/region topology labels. Require `--cloud-provider=external` on kubelet.
- **Metal provider** (`manual`) — no cloud API, no CCM. `--cloud-provider=external` is not set; the uninitialized taint is never applied.

## Decision

Provider type determines CCM behaviour at bootstrap:

| Provider | `--cloud-provider=external` | CCM installed | Pre-Flux step |
|---|---|---|---|
| `hcloud` | ✅ | ✅ hcloud CCM | Cilium → hcloud CCM → flux bootstrap |
| `manual` (metal) | ❌ | ❌ | Cilium → flux bootstrap |

The pre-Flux component list is driven by a provider `group_vars` variable (e.g. `bootstrap_pre_flux_components`). Ansible's `bootstrap-flux.yml` iterates it; metal omits CCM entirely.

### hcloud CCM configuration (reference implementation)

Deployed via Helm with `networking.enabled: false`. Active roles:

- **Kubernetes cloud controller** — node lifecycle, `providerID = hcloud://<id>`, `InternalIP` from private network, `topology.kubernetes.io/zone` / `region` labels, clears uninitialized taint.
- **CSI prerequisite** — Hetzner CSI requires `providerID` on each node to attach Cloud Volumes.

Disabled roles:
- **Route controller** (`networking.enabled: false`) — not needed; CNI uses VXLAN.
- **LB controller** — not needed; ingress uses host-network Envoy Gateway (see ADR-004).

## Consequences

- CCM runs as a plain Helm release, outside Flux. It is not reconciled by Flux and must be upgraded via playbook or manually.
- Adding a new cloud provider to the bootstrap template requires defining its CCM chart/values in the provider `group_vars` and adding it to `bootstrap_pre_flux_components`.
- Components that cannot tolerate the uninitialized taint must not be placed in the Ansible pre-Flux layer — they belong in Flux, where the taint is already cleared.
