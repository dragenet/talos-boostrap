# ADR-005: Hetzner CSI driver for block storage

**Status:** Accepted

## Context

Workloads need persistent block storage backed by network-attached volumes. Hetzner Cloud provides Cloud Volumes (block devices attachable to servers in the same location) and a CSI driver to provision them dynamically as Kubernetes PVCs.

## Decision

Deploy the **Hetzner CSI driver** (`hcloud-csi`) as an infrastructure controller (Flux `infrastructure/controllers`). It provisions `hcloud-volumes` StorageClass-backed PVCs as Hetzner Cloud Volumes attached to the node running the workload.

Requires the hcloud CCM to be running first — the CSI driver uses `node.spec.providerID` to identify which Hetzner server to attach the volume to (see cluster-bootstrap ADR-005).

## Consequences

- Volumes are **location-scoped** (`nbg1`) — a volume cannot be attached to a node in a different Hetzner location. This is a non-issue for the single-DC topology (see kube1 ADR-002) but would block a multi-DC expansion.
- Volumes are **single-attach** (ReadWriteOnce) — no shared block access across pods on different nodes. Workloads needing shared storage use a different mechanism (e.g. an object store).
- Hetzner Cloud Volumes are billed per GB provisioned (verify current pricing live at hetzner.com). Unused PVCs still incur cost.
- Latency-sensitive workloads (databases, message queues) prefer local storage — see cluster-bootstrap ADR-009 (OpenEBS LocalLV LVM) for the low-latency tier.
