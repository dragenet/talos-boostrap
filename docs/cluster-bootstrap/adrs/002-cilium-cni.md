# ADR-002: Cilium as the default CNI (VXLAN, kube-proxy replacement, L7 proxy disabled)

**Status:** Accepted

## Context

A CNI is required for pod networking. The choice of CNI also determines whether kube-proxy runs, whether L7 policy is handled in-CNI, and what the ingress/Gateway API controller is.

CNI is a **selectable, Talos-coupled choice** (`network.cni: cilium | flannel`, see ADR-011/012/013): `cilium` sets `cni.name: none` + `proxy.disabled: true` in the Talos patch and installs Cilium; `flannel` leaves the Talos default (Flannel + kube-proxy) and installs no CNI add-on. This ADR records the **default** (`cilium`) and its configuration.

## Decision

Default to **Cilium** with:
- `routingMode: vxlan` (default tunnel mode)
- `kubeProxyReplacement: true` — Cilium's eBPF replaces kube-proxy entirely for Service load-balancing
- `l7Proxy: false` — Cilium's built-in Envoy sidecar is disabled; L7 routing and policy are owned by Envoy Gateway (see ADR-004)
- `gatewayAPI.enabled: false` — Cilium's Gateway API controller is disabled; Envoy Gateway owns all Gateway API resources

## Consequences

- **VXLAN over native routing:** The Hetzner private network is L3-routed; `autoDirectNodeRoutes` does not work due to Hetzner's injected gateway (cilium#24777). Native routing requires the hcloud CCM route controller to program per-node pod CIDR routes. VXLAN is fabric-agnostic and simpler; the ~50-byte MTU overhead is an acceptable trade for this topology.
- **No Cilium L7 NetworkPolicy** — HTTP-aware policy (`toFQDNs`, L7 rules) is unavailable. All network policy is L3/L4.
- **No Cilium Hubble L7 flows** — Hubble visibility is L3/L4 only.
- `l7Proxy=false` is a hard prerequisite for `gatewayAPI.enabled=false`; the two travel together.
- **Choosing `flannel` is a real downgrade:** no Cilium NetworkPolicy, no Hubble, no eBPF kube-proxy replacement or LB. The rest of the stack (Envoy Gateway, cert-manager, CCM, CSI) is CNI-agnostic and runs on either. `flannel` is the "basic networking only" path; `cilium` is the full-feature default.
- Chose Cilium over Calico (iptables-heavy, no kube-proxy replacement by default) and Weave (abandoned); Flannel is kept as the selectable lightweight alternative rather than rejected outright.
