# Bootstrap Recovery Notes

This note captures the bootstrap-only fixes that were added after the first
cluster bring-up exposed a few environment-specific failure modes.

## Hetzner kubeconfig endpoint

Talos generates `infra/talos/secrets/kubeconfig` against the private cluster
VIP (`10.0.0.10:6443`). That endpoint is not reachable from the local machine,
so `bootstrap-talos.yml` now patches the kubeconfig server URL to the first
hybrid node's public IP after fetch. This is Hetzner-specific and is wired via
`infra/ansible/inventories/providers/hcloud/group_vars/all/kubeconfig.yml`.

## Pod Security Admission overrides

Talos enforces PSA `baseline` cluster-wide. The namespaces for Cilium,
hcloud CCM, OpenEBS CSI, and Envoy Gateway need `pod-security.kubernetes.io/enforce=privileged`
because their pods use `hostNetwork`, `hostPath`, or privileged containers.
Those labels are now part of the Flux-managed namespace manifests so Flux does
not strip them on reconciliation.

## OpenEBS disk layout

On the example cluster's CX33 disks, Talos EPHEMERAL initially consumed the whole system
disk and left no free space for the OpenEBS raw volume. The Talos config now
emits an `EPHEMERAL` `VolumeConfig` cap before the OpenEBS `RawVolumeConfig`,
so a fresh rebuild leaves room for `/dev/disk/by-partlabel/r-openebs-lvm`.

Important: this is a **rebuild-time** fix, not a reboot-time fix. Existing
partitions are not shrunk in place.

## Gateway API / Envoy CRDs

The Envoy Gateway CRD chart exceeded Helm's release Secret size limit.
Instead of a HelmRelease, the rendered CRDs are committed as raw YAML under
`flux/infrastructure/controllers/_components/gateway-api/standard/` and applied
through the normal Flux Kustomization.
