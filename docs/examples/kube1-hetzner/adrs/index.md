---
title: kube1 Hetzner — Architecture Decision Records
description: ADR log for the kube1 Hetzner Cloud example cluster — provider-specific decisions layered on top of the generic template ADRs.
type: index
audience: [user, contributor]
tags: [adr, kube1, hetzner]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - ../index.md
  - ../../../adr/index.md
---

# kube1 Hetzner — Architecture Decision Records

This section logs Architecture Decision Records specific to the **kube1** Hetzner Cloud example cluster. These decisions are layered on top of the generic template ADRs (see [generic template ADRs](../../../adr/index.md)) and capture Hetzner-specific choices around topology, networking, TLS, and storage.

## ADR Log

| Number | Title | Status |
|--------|-------|--------|
| [0001](0001-hetzner-cloud-provider.md) | ADR-001: Hetzner Cloud as the infrastructure provider | accepted |
| [0002](0002-single-dc-hybrid-topology.md) | ADR-002: Single DC, three hybrid nodes | accepted |
| [0003](0003-three-plane-network-ingress.md) | ADR-003: Three-plane network ingress model | accepted |
| [0004](0004-tls-lets-encrypt-cloudflare.md) | ADR-004: TLS via Let's Encrypt + Cloudflare DNS-01 on dragenet.dev | accepted |
| [0005](0005-hetzner-csi.md) | ADR-005: Hetzner CSI driver for block storage | accepted |
