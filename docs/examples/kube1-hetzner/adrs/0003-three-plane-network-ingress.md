---
title: "ADR-003: Three-plane network ingress model"
description: Traffic is split across three planes: public HTTPS via DNS-RR+Envoy, zero-trust via cloudflared tunnel, and admin TCP/UDP via Tailscale.
type: adr
audience: [contributor, ai]
tags: [adr, networking, ingress, tailscale, envoy]
status: accepted
created: 2026-07-01
updated: 2026-07-01
---

# ADR-003: Three-plane network ingress model

**Status:** Accepted

## Context

The cluster needs to accept traffic from three distinct sources with different trust levels, protocols, and exposure requirements: public internet HTTP/S, authenticated zero-trust HTTP/S, and admin/arbitrary TCP+UDP. All routing is Gateway API — Kubernetes Ingress is not used.

## Decision

Traffic is split across three independent planes:

| Plane | Traffic | Mechanism |
|---|---|---|
| **Public HTTPS** | Public-facing web services | DNS round-robin across node public IPs (or a Floating IP) → Envoy Gateway in host-network mode (ports 80/443 on each node) |
| **Zero-trust HTTPS** | Private/internal web apps | `cloudflared` in-cluster (outbound tunnel) → Envoy Gateway ClusterIP. Zero open ports, zero public IPs. |
| **Admin + TCP/UDP** | `talosctl`, `kubectl`, arbitrary TCP/UDP services | Tailscale (`siderolabs/tailscale` extension) — nodes join the tailnet; access scoped by Tailscale ACLs |

## Consequences

- **No Hetzner Cloud Load Balancer needed** for ingress — the CCM LB controller is not deployed. Saves ~€6/mo per LB and removes a managed dependency.
- **No Cilium L2 announcements** — L2 ARP-based announcement doesn't cross Hetzner's L3-routed private network and cannot deliver public ingress. Dropped entirely.
- **Host-network Gateway on public nodes** — ports 80/443 must be opened on the hcloud firewall. All three nodes serve as ingress replicas; DNS TTL governs client failover.
- **Floating IP alternative:** a Hetzner Floating IP (~€3/mo IPv4, verify live) gives a single stable public IP but is active/passive (one node at a time) and requires an in-cluster controller to reassign it via the hcloud API on node failure. DNS-RR is the default; adopt FIP if a stable public IP becomes a requirement.
- **Tailscale covers the admin plane** — `talosctl` and `kubectl` use Tailscale IPs; each node's Tailscale IP must be added to `cluster_talos_cert_sans` and `cluster_k8s_cert_sans`. Once Tailscale is operational, Talos API and Kubernetes API firewall rules can be closed to the public internet.
- **Cloudflare Tunnel is a runtime dependency** for the zero-trust plane — if the tunnel is down, those services are unreachable. Acceptable for a homelab.
