---
title: "ADR-001: Hetzner Cloud as the infrastructure provider"
description: Hetzner Cloud (nbg1, CX33 instances) is chosen as the sole kube1 provider for its affordability, API, and European DC availability.
type: adr
audience: [contributor, ai]
tags: [adr, hetzner, infrastructure, provider]
status: accepted
created: 2026-07-01
updated: 2026-07-01
---

# ADR-001: Hetzner Cloud as the infrastructure provider

**Status:** Accepted

## Context

A cloud provider is needed to run the cluster. Key requirements: affordable dedicated VMs, a private L3 network for inter-node traffic, an API suitable for automated provisioning, and a European data centre for latency.

## Decision

Use **Hetzner Cloud** (`nbg1`, Nuremberg) as the sole provider for kube1. Nodes run on **CX33** instances (verify current specs and pricing live at hetzner.com — prices changed significantly in April 2026). Storage uses Hetzner Cloud Volumes via the CSI driver. The private network is `kube1` (`10.0.0.0/16`, subnet `10.0.0.0/24`).

## Consequences

- The hcloud dynamic inventory plugin (`hetzner.hcloud.hcloud`) drives Ansible; nodes are discovered by label, not hard-coded IPs.
- The hcloud API token is SOPS-encrypted at `inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml`.
- hcloud firewall rules restrict Talos API (50000) and Kubernetes API (6443) access. Firewall rules are managed by `provision-infra.yml`; the `hcloud_firewall_open_access` flag toggles between IP-scoped and open-to-internet modes.
- Hetzner's private network is **L3-routed** — ARP does not propagate between subnets or DCs. This is the root constraint that pins the cluster to a single DC (see ADR-002).
