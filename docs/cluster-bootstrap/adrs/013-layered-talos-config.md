# ADR-013: Layered Talos config — structured helpers + raw fragments

**Status:** Superseded by ADR-016

## Context

Talos machine config (disk partitioning, network) ranges from trivial (single NIC + VIP, one data disk) to complex (bonds, VLANs, multiple NICs, mixed disk layouts). The config surface must cover the common case ergonomically without ever capping expressiveness — we must not lose what hand-written Talos patches can already do.

## Decision

A **two-layer** model, both rendered into the machine config and merged via talosctl's native `--config-patch` (later layers win):

- **Layer 1 — structured helpers** in `cluster.yaml` for the common case:
  - `talos.install` (disk / `diskSelector`);
  - `storage.volumes[]` (data volumes);
  - `network` (VIP, default interface).
- **Layer 2 — raw Talos fragments** for anything else, verbatim:
  - `talos.patches.{all,controlplane}` (cluster-wide / per-role);
  - `nodes[].patch` (per-node).

Layer 2 is a strict superset of the current `.j2` patches, so nothing is lost.

### Disk (Talos v1.13)
Use **`UserVolumeConfig`** (formatted+mounted) / **`RawVolumeConfig`** (raw block device); `machine.disks` is deprecated. Volumes are selected by a CEL **`diskSelector`** + `minSize`/`maxSize`, so "separate disk", "one disk / two partitions", and "Talos-only" are all just selector + free-space differences. **OpenEBS LVM needs a raw device** (`RawVolumeConfig` or whole disk), not a mounted UserVolume — there's known friction (talos#11847); render validates `openebs ⇒ raw volume`.

### Network (Talos v1.13)
Match interfaces by **`deviceSelector`** (names aren't stable), not `eth0`/`eth1`. Bonds use `bond.deviceSelectors` with **`permanentAddr`** (the MAC changes once enslaved). VLANs stack via `vlans[]`; the VIP can sit on a specific VLAN. Multi-NIC/bond config is inherently **per-node** (MACs differ) → it lives in `nodes.yaml`.

### Scope split
- cluster-wide shape (which VLANs, bond mode, VIP, default disk) → `cluster.yaml`;
- per-node hardware (MACs, addresses, physical NICs/disks) → `nodes.yaml`.

## Consequences

- The renderer composes: `talosctl gen` base + Layer-1 helper output + Layer-2 raw patches (`all`/`controlplane`/per-node), later winning. No bespoke merge engine.
- Rendered machine configs embed the cluster CA + keys → they stay **gitignored** (`infra/talos/secrets/`), regenerated locally from committed templates + the Talos secrets bundle. This is the deliberate carve-out from "everything is committed."
- A disk/volume device path is a Talos↔Flux coupling point — it flows into `cluster-vars` so the OpenEBS config references the same device the Talos patch prepared.

## Sources
- [Talos v1.13 User Volumes](https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/storage-and-disk-management/disk-management/user)
- [talos#11847 — UserVolume on LVM](https://github.com/siderolabs/talos/issues/11847)
- [Talos Network Device Selector](https://www.talos.dev/v1.11/talos-guides/network/device-selector/)
