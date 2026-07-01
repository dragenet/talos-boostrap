---
title: Disaster Recovery
description: Bootstrap-phase recovery notes covering kubeconfig endpoint patching, PSA overrides, OpenEBS disk layout, and Gateway API CRD sizing.
type: guide
audience: [operator]
tags: [disaster-recovery, bootstrap, talos]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - getting-started.md
  - storage.md
  - ingress-and-tls.md
---

# Disaster Recovery

This guide captures bootstrap-only fixes and known failure modes that were identified during initial cluster bring-up. It documents the workarounds now permanently baked into the template.

## Kubeconfig endpoint patching

Talos generates `infra/talos/secrets/kubeconfig` against the private cluster
VIP (`10.0.0.10:6443`). That endpoint is not reachable from the local machine,
so `bootstrap-talos.yml` now patches the kubeconfig server URL to the first
hybrid node's public IP after fetch. This is provider-specific (currently wired
for hcloud) and is configured via
`infra/ansible/inventories/providers/hcloud/group_vars/all/kubeconfig.yml`.

## Pod Security Admission overrides

Talos enforces PSA `baseline` cluster-wide. The namespaces for Cilium,
hcloud CCM, OpenEBS CSI, and Envoy Gateway need `pod-security.kubernetes.io/enforce=privileged`
because their pods use `hostNetwork`, `hostPath`, or privileged containers.
Those labels are now part of the Flux-managed namespace manifests so Flux does
not strip them on reconciliation.

## OpenEBS disk layout

On clusters where Talos EPHEMERAL initially consumed the whole system
disk, no free space was left for the OpenEBS raw volume. The Talos config now
emits an `EPHEMERAL` `VolumeConfig` cap before the OpenEBS `RawVolumeConfig`,
so a fresh rebuild leaves room for `/dev/disk/by-partlabel/r-openebs-lvm`.

Important: this is a **rebuild-time** fix, not a reboot-time fix. Existing
partitions are not shrunk in place. See [Storage](storage.md) for the full
operational runbook.

## Gateway API / Envoy CRDs

The Envoy Gateway CRD chart exceeded Helm's release Secret size limit.
Instead of a HelmRelease, the rendered CRDs are committed as raw YAML under
`flux/infrastructure/controllers/_components/gateway-api/standard/` and applied
through the normal Flux Kustomization. See [Ingress and TLS](ingress-and-tls.md)
for the full operational runbook.
