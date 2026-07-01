---
title: Layered Talos Config
description: How Talos machine configs are composed from base configs, structured helpers, and raw patch fragments merged in priority order.
type: concept
audience: [user, contributor]
tags: [talos, config]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - config-compiler.md
  - architecture-overview.md
  - ../adr/0013-layered-talos-config.md
---

# Layered Talos Config

Talos machine configuration is not written as a single monolithic file. Instead, the render assembles it from multiple layers — each owned by the appropriate level of the hierarchy — and merges them using `talosctl`'s native `--config-patch` mechanism, where later layers win.

## Why Layering?

A bare-metal single-node lab and a cloud-deployed multi-node cluster share most of their Talos config: bootstrap settings, kubelet arguments, extension lists. Only a small portion varies: the disk selector, the network interface, per-node MACs. Without layering, every cluster would need a full copy of every config, making template updates expensive and error-prone.

Layering separates what changes from what doesn't: shared fragments live in template-owned bases; cluster-specific and node-specific details live in user-owned overlays.

## The Two-Layer Model (ADR-013)

The render composes two layers for each node's machine config, later winning:

### Layer 1 — Structured Helpers

`cluster.yaml` provides ergonomic keys for the common configuration surface:

- `talos.install` — install disk selection (disk path or `diskSelector` expression).
- `storage.volumes[]` — data volume definitions (`UserVolumeConfig` for formatted+mounted, `RawVolumeConfig` for raw block devices).
- `network` — VIP, default interface matching via `deviceSelector`.

These cover the majority of clusters without requiring manual Talos patch authoring.

### Layer 2 — Raw Talos Fragments

For anything that the structured helpers do not cover, Layer 2 accepts verbatim Talos patch fragments:

- `talos.patches.all` — applied to every node in the cluster.
- `talos.patches.controlplane` — applied to control-plane nodes only.
- `nodes[].patch` — applied to a single specific node.

Layer 2 is a strict superset of hand-written Talos patches. Nothing expressible in a Talos patch file is blocked.

## Scope Split: cluster.yaml vs nodes.yaml

The config surface is split along a deliberate boundary:

| File | Contains |
|---|---|
| `cluster.yaml` | Cluster-wide shape: which VLANs, bond mode, VIP address, default disk selector |
| `nodes.yaml` | Per-node hardware: MAC addresses, physical NIC names, per-node patches |

Multi-NIC and bond configurations are inherently per-node (because MAC addresses differ between nodes) and therefore live in `nodes.yaml`. Network interfaces are matched by `deviceSelector` (not `eth0`/`eth1`) because interface names are not stable across reboots in Talos.

## Where Layers Live in the Repo

```
config/
  talos/                          # Template-owned shared bases and component fragments
  defaults/                       # Template-owned defaults merged into cluster.yaml
  clusters/<cluster>/
    cluster.yaml                  # Layer-1 structured helpers + Layer-2 cluster-wide patches
    nodes.yaml                    # Per-node hardware details and per-node patches
    talos/                        # User-authored Talos overlay patches for this cluster
```

The render reads these inputs, produces Talos config patches, and passes them to `talosctl` via `--config-patch`. The merge order is: `talosctl gen` base → Layer-1 helper output → Layer-2 raw patches (`all` → `controlplane` → per-node). Later layers win on conflict.

## Disk and Volume Notes

Talos v1.13 uses `UserVolumeConfig` (formatted and mounted) and `RawVolumeConfig` (raw block device). The older `machine.disks` API is deprecated. Volumes are selected by a CEL `diskSelector` expression with optional `minSize`/`maxSize` bounds, so "a separate data disk", "two partitions on one disk", and "Talos-only" are all expressed as selector + free-space differences.

**OpenEBS LVM** requires a raw block device (`RawVolumeConfig` or a whole disk), not a mounted `UserVolume`. The render validates this: `openebs feature enabled ⇒ a raw volume must be declared`.

Disk and volume device paths are a Talos–Flux coupling point: the render includes them in the `cluster-vars` ConfigMap so that the OpenEBS configuration in Flux references the same device that the Talos patch prepared.

## Network Notes

Match network interfaces with `deviceSelector` expressions, not by name. For bond configurations, use `bond.deviceSelectors` with `permanentAddr` (the hardware MAC address, which changes once the interface is enslaved to a bond). VLANs stack via `vlans[]`; the VIP can be placed on a specific VLAN.

## Further Reading

- [Config Compiler](config-compiler.md) — how the render assembles these layers
- [Architecture Overview](architecture-overview.md) — where Talos config fits in the overall system
- [ADR-013: Layered Talos Config](../adr/0013-layered-talos-config.md)
