---
title: Architecture Decision Records
description: Log of all generic template ADRs — numbered decisions that shaped the architecture of the Talos Kubernetes bootstrap template.
type: index
audience: [contributor, operator, ai]
tags: [adr, architecture, decisions]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - ../index.md
  - ../contributing/writing-adrs.md
---

# Architecture Decision Records

This section logs all generic template Architecture Decision Records (ADRs). Each ADR records a significant design decision: its context, the decision made, considered alternatives, and consequences.

ADRs are immutable once accepted. Superseded decisions link to the ADR that replaced them.

For instance-specific ADRs (e.g. for the kube1 Hetzner example), see [examples/kube1-hetzner/adrs/](../examples/kube1-hetzner/adrs/).

## ADR Log

| Number | Title | Status |
|--------|-------|--------|
| [0001](0001-talos-linux.md) | ADR-001: Talos Linux as the cluster OS | accepted |
| [0002](0002-cilium-cni.md) | ADR-002: CNI selection and Cilium configuration | accepted |
| 0003 | _(not used — number reserved)_ | — |
| [0004](0004-envoy-gateway-l7.md) | ADR-004: Envoy Gateway as the sole Gateway API implementation | accepted |
| [0005](0005-cloud-controller-manager-pattern.md) | ADR-005: Cloud controller manager — tri-state enablement and provider resolution | accepted |
| [0006](0006-fluxcd-gitops.md) | ADR-006: FluxCD for GitOps | accepted |
| [0007](0007-ansible-portable-bootstrap.md) | ADR-007: Ansible as a portable, provider-agnostic cluster bootstrap layer | accepted |
| [0008](0008-cert-manager.md) | ADR-008: cert-manager for TLS certificate lifecycle | accepted |
| [0009](0009-openebs-local-lv-lvm.md) | ADR-009: OpenEBS LocalLV LVM for low-latency local storage | accepted |
| [0010](0010-ansible-flux-provider-handoff.md) | ADR-010: Ansible-to-Flux provider handoff — pre-Flux layer and provider overlays | accepted |
| [0011](0011-config-compiler.md) | ADR-011: Config compiler — single input, deterministic render, one writer per file | accepted |
| [0012](0012-feature-catalog.md) | ADR-012: Feature catalog and taxonomy | accepted |
| [0013](0013-layered-talos-config.md) | ADR-013: Layered Talos config — structured helpers + raw fragments | superseded |
| [0014](0014-node-discovery-dynamic-inventory.md) | ADR-014: Node discovery via inventory, not filesystem IPC | accepted |
| [0015](0015-flux-bootstrap-auth.md) | ADR-015: Flux bootstrap authentication modes and secret persistence | accepted |
| [0016](0016-talos-config-boundary.md) | ADR-016: Talos configuration boundary — Talos-shaped patches, Ansible as compiler | accepted |
| [0017](0017-shared-overlay-ownership.md) | ADR-017: Shared overlay ownership model across Talos and Flux | accepted |
| [0018](0018-flux-entrypoint-subdirectories.md) | ADR-018: Flux cluster entrypoint subdirectories | accepted |

## How to write an ADR

See [`contributing/writing-adrs.md`](../contributing/writing-adrs.md) for the ADR template and writing guidance.
