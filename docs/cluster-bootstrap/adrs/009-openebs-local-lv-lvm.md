# ADR-009: OpenEBS LocalLV LVM for low-latency local storage

**Status:** Accepted — **selectable, Talos-coupled feature** (`features.openebs`; see ADR-012). Enabling it renders both the Flux HelmRelease *and* the Talos disk/volume config (ADR-013), so it requires an Ansible render + bootstrap, not a pure-git toggle.

## Context

Network-attached block storage (e.g. Hetzner CSI) adds latency from the network hop and the storage backend. Latency-sensitive workloads (databases, message queues) benefit from storage that lives on the node's own disk. A second storage tier on local disks is needed alongside the networked tier.

## Decision

Deploy **OpenEBS LocalLV LVM** as the local storage tier. OpenEBS manages an LVM Volume Group on a dedicated disk partition on each node and provisions PVCs as LVM Logical Volumes directly on that VG. This gives local-disk performance with Kubernetes-native PVC semantics.

The block device for the LVM VG is provisioned at **Talos bootstrap time** via a `RawVolumeConfig` (raw block device — LVM needs an unformatted device, not a mounted `UserVolume`; see ADR-013 and talos#11847) selected by `diskSelector`. This is a required step in `bootstrap-talos.yml` / the `talos_config` role, not a post-install manual step. (`machine.disks` is deprecated in Talos v1.13.)

## Consequences

- **Local storage is not replicated** — if the node is lost, the data on its local PVCs is lost. Workloads using this tier must handle their own replication (e.g. application-level replication, regular backups) or accept the risk.
- **Pod is pinned to the node** — LocalLV PVCs are node-affinity-bound; a pod using a local PVC cannot be rescheduled to a different node. Plan for this in workload design.
- **Talos disk layout must be planned before first boot.** The volume's `diskSelector`/`minSize` is set in `cluster.yaml` `storage.volumes`; changing the layout later requires wiping and re-provisioning the node. Size the LVM device with headroom.
- **Two storage classes in the cluster:** `hcloud-volumes` (network-attached, portable, replicated by Hetzner) and `openebs-lvm-localpv` (local, fast, node-pinned). Workload authors choose based on latency vs durability requirements.
- Chose OpenEBS LocalLV LVM over `hostPath` (no PVC lifecycle management), `local-path-provisioner` (no LVM thin-provisioning or snapshot support), and Rook/Ceph (heavy, full distributed storage stack — overkill for a 3-node homelab).
