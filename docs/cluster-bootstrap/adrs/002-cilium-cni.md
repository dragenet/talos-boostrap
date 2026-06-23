# ADR-002: CNI selection and Cilium configuration

**Status:** Accepted

## Context

A CNI is core bootstrap configuration: pod networking must exist before most workloads schedule, and the choice determines whether kube-proxy runs, whether L7 policy is handled in-CNI, and what ingress/Gateway API controller is used. Because it is foundational, CNI is selected via top-level cluster config (`cni.name`), not via the optional `features` catalog (ADR-012).

The template supports two CNI modes:

- `flannel` — the Talos default, works out of the box, uses kube-proxy.
- `cilium` — an eBPF-based CNI that replaces kube-proxy and supports NetworkPolicy.

This ADR records the selection mechanism and the Cilium configuration when it is chosen.

## Decision

- `cni.name` is a core cluster config key with allowed values `flannel` (default) and `cilium`.
- `cni.name: flannel` leaves the Talos default CNI (Flannel) and kube-proxy in place; no add-on is installed.
- `cni.name: cilium` renders a Talos patch that disables the bundled CNI (`cni.name: none` and `proxy.disabled: true`) and installs Cilium via Helm into the `cilium-system` namespace.
- When Cilium is selected, configure it as:
  - `routingMode: vxlan`
  - `kubeProxyReplacement: true`
  - `l7Proxy: false`
  - `gatewayAPI.enabled: false`

Cilium is installed by Ansible as a pre-Flux component (ADR-010) and then adopted by Flux after bootstrap using the same Helm release name, namespace, and values.

## Consequences

- **VXLAN over native routing:** The Hetzner private network is L3-routed; `autoDirectNodeRoutes` does not work due to Hetzner's injected gateway (cilium#24777). Native routing would require the hcloud CCM route controller to program per-node pod CIDR routes. VXLAN is fabric-agnostic and simpler; the ~50-byte MTU overhead is an acceptable trade for this topology.
- **No Cilium L7 NetworkPolicy** — HTTP-aware policy (`toFQDNs`, L7 rules) is unavailable. All network policy is L3/L4.
- **No Cilium Hubble L7 flows** — Hubble visibility is L3/L4 only.
- `l7Proxy=false` is a hard prerequisite for `gatewayAPI.enabled=false`; the two travel together.
- **Flannel is the lightweight path:** no Cilium NetworkPolicy, no Hubble, no eBPF kube-proxy replacement. The rest of the stack (Envoy Gateway, cert-manager, CCM, CSI) is CNI-agnostic and runs on either. `flannel` is the "basic networking only" path; `cilium` is the full-featured selectable alternative.
- Cilium runs in its own namespace (`cilium-system`), not `kube-system`, consistent with the bootstrap component namespace convention.
- We rejected Calico (iptables-heavy, no kube-proxy replacement by default) and Weave (abandoned). Flannel is kept as the default lightweight alternative rather than rejected outright.

> **Implementation status (2026-06-23):** Implemented. The `cni.name` schema migration is complete (`features.cni` is no longer used). The `flux_bootstrap` role installs Cilium pre-Flux when `cni.name: cilium`, and the matching `HelmRelease` under `flux/infrastructure/controllers/_components/cilium/` adopts the release after `flux bootstrap`.
