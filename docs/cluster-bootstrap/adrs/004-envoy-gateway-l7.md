# ADR-004: Envoy Gateway as the sole Gateway API implementation

**Status:** Accepted — **default** ingress; `features.ingress` is pick-one (`envoy-gateway | traefik | none`, pure-Flux; see ADR-012). When enabled, Envoy Gateway is the *sole* Gateway API implementation; Traefik is the selectable alternative.

## Context

Gateway API is the sole routing abstraction — Kubernetes Ingress is not used. A Gateway API controller is needed for L7 HTTP/HTTPS routing (and optionally L4 TCPRoute/UDPRoute for non-Tailscale TCP/UDP). Cilium ships a built-in Gateway API controller backed by its Envoy sidecar, but enabling it requires `l7Proxy=true` — which couples L7 routing to the CNI and makes Cilium's Envoy a shared dependency for both policy and routing.

## Decision

Deploy **Envoy Gateway** as the sole Gateway API controller. Cilium's `l7Proxy` and `gatewayAPI` are both disabled (see ADR-002). Envoy Gateway runs its own managed Envoy proxies and owns all `GatewayClass`, `HTTPRoute`, `GRPCRoute`, `TCPRoute`, and `UDPRoute` resources.

The Envoy proxy data plane is exposed via host-network mode (configured via the `EnvoyProxy` CRD on the Gateway) so that DNS-RR and Cloudflare Tunnel can reach it without a LoadBalancer-type Service.

## Consequences

- **Clean separation:** Cilium owns L3/L4; Envoy Gateway owns L7. L7 policy (AuthorizationPolicy, SecurityPolicy) lives in Envoy Gateway CRDs, not Cilium NetworkPolicy.
- **No LoadBalancer Service required** — `EnvoyProxy` spec patches the proxy Deployment to use host networking; no CCM LB controller, no MetalLB.
- **Gateway API CRDs:** install the **experimental** channel if `TCPRoute`/`UDPRoute` are needed; standard channel suffices for HTTP/GRPC-only workloads.
- Chose Envoy Gateway over Traefik (non-standard CRDs, not Gateway API native), ingress-nginx (Ingress API, not Gateway API), and Cilium Gateway (couples L7 routing lifecycle to CNI upgrades).
