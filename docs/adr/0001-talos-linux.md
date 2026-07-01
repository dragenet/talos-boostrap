---
title: "ADR-001: Talos Linux as the cluster OS"
description: Talos Linux is chosen as the immutable, API-driven OS for all cluster nodes, eliminating SSH and mutable state.
type: adr
audience: [contributor, ai]
tags: [adr, talos, os, security]
status: accepted
created: 2026-07-01
updated: 2026-07-01
---

# ADR-001: Talos Linux as the cluster OS

**Status:** Accepted

## Context

Kubernetes nodes need an OS. Traditional distributions (Ubuntu, Debian, RHEL) carry SSH, package managers, mutable filesystems, and general-purpose tooling that broaden the attack surface and require ongoing OS-level maintenance. Immutable, purpose-built alternatives exist.

## Decision

Use **Talos Linux** as the OS on all nodes. Talos is API-driven (gRPC), has no SSH, a read-only root filesystem, and a minimal footprint — the only interface is `talosctl`. Node configuration is declarative (machine config YAML), applied idempotently. Upgrades are atomic image swaps, not package updates.

## Consequences

- No SSH access to nodes; debugging via `talosctl dmesg`, `talosctl logs`, `talosctl console`.
- Machine configs are generated via `talosctl gen config` and must be kept out of git (they contain the cluster CA and keys).
- System extensions (e.g. Tailscale, QEMU guest agent) are baked into the installer image at build time via the [Talos Image Factory](https://factory.talos.dev), not installed at runtime. Adding/removing an extension requires a new schematic ID and a node image upgrade.
- Chose Talos over k3os (abandoned), Flatcar (mutable enough to be Ubuntu-like), and RKE2-on-Ubuntu (SSH surface, manual patching).
