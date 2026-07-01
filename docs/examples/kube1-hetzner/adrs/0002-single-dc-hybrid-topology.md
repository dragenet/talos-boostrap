---
title: "ADR-002: Single DC, three hybrid nodes"
description: Three hybrid (control-plane+worker) nodes in a single Hetzner DC; mandatory for the Talos ARP-based VIP which cannot cross L3 DC boundaries.
type: adr
audience: [contributor, ai]
tags: [adr, topology, hetzner, networking]
status: accepted
created: 2026-07-01
updated: 2026-07-01
---

# ADR-002: Single DC, three hybrid nodes

**Status:** Accepted

## Context

The cluster needs control-plane HA (etcd quorum) and the ability to run workloads. Node roles can be separated (dedicated control-plane + worker) or combined. The Talos VIP (ARP-based floating Kubernetes API endpoint) has a hard network constraint.

## Decision

Run **three hybrid nodes** (control-plane + worker combined) in a **single Hetzner DC** (`nbg1`). All nodes run etcd, schedule workloads, and share the Talos native VIP (`10.0.0.10`).

Node naming: `kube1-hb1`, `kube1-hb2`, `kube1-hb3`. Private IPs assigned deterministically from `10.0.0.11` onwards.

## Consequences

- **Single DC is mandatory for the Talos VIP.** The VIP uses ARP gratuitous announcements (L2). Hetzner private networks are L3-routed — ARP does not propagate across DC boundaries. Multi-DC would require a Hetzner Cloud Load Balancer as the API endpoint, adding cost and a managed dependency.
- **Three nodes = minimum viable etcd quorum.** Tolerates one node failure. Scaling to 5 nodes improves fault tolerance but is not planned.
- **Hybrid nodes** simplify operations at this scale — no separate control-plane taint/toleration management, fewer node types to reason about. If the cluster grows to the point where control-plane resource isolation matters, a role split can be introduced.
- `allowSchedulingOnControlPlanes: true` is applied to hybrid nodes; pure control-plane nodes (if ever added) keep the default NoSchedule taint.
