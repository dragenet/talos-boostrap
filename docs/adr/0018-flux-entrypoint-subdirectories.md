---
title: "ADR-018: Flux cluster entrypoint subdirectories"
description: Cluster entrypoints are split into bootstrap-owned flux-system/, infrastructure/, and apps/ subdirectories to support standalone infrastructure roots.
type: adr
audience: [contributor, ai]
tags: [adr, flux, gitops, clusters]
status: accepted
created: 2026-07-03
updated: 2026-07-03
---

# ADR-018: Flux cluster entrypoint subdirectories

**Status:** Accepted

## Context

The repo's Flux cluster entrypoints started as a flat directory containing the bootstrap-owned `flux-system/` content, the foundational `infrastructure.yaml` pair, and flat per-app `*.yaml` Kustomization CRs. That shape was sufficient for one infrastructure chain and a few apps, but it does not scale cleanly once we add additional standalone infrastructure roots that are not part of the template-managed controller/config pipeline.

The new kube1 DNS deployment is the first such case: it needs its own standalone Flux `Kustomization` CR and its own hand-authored manifests, but it should still live alongside the rest of the cluster entrypoint rather than being merged into the template-owned infrastructure catalog.

## Decision

Each `flux/clusters/<cluster>/` directory is split into three explicit subdirectories:

- `flux-system/` for bootstrap-owned Flux manifests;
- `infrastructure/` for the core infrastructure chain plus any additional standalone infrastructure Kustomization CRs;
- `apps/` for per-application Kustomization CRs.

The existing `infrastructure.yaml` file becomes `infrastructure/core.yaml` and continues to hold the `infrastructure-controllers` and `infrastructure-configs` CRs unchanged. Additional cluster-specific infrastructure CRs are added as separate files under `infrastructure/`. Per-app CRs move from flat `<app>.yaml` files into `apps/<app>.yaml`.

Flux still auto-sweeps the cluster root recursively from the bootstrap-owned `flux-system` Kustomization, so no `kustomization.yaml` is added at the cluster root or subdirectory roots.

This decision refines ADR-006 and ADR-017 without changing the underlying Flux bootstrap model.

## Consequences

- Cluster entrypoints now have a stable place for extra infrastructure roots without overloading the template-owned controller/config chain.
- Per-app CRs are separated from infrastructure CRs, which makes cluster ownership and reconciliation intent clearer.
- The layout is slightly deeper, but the new structure matches how Flux already observes the repository.
- New cluster-entrypoint docs and examples must describe the `infrastructure/` and `apps/` subdirectories instead of the old flat layout.

## Rejected alternatives

- **Keep the flat cluster root.** Rejected because it becomes ambiguous once multiple standalone infrastructure roots exist.
- **Move standalone infrastructure into the template-owned controller/config pipeline.** Rejected because kube1 DNS is intentionally custom and should reconcile independently.
- **Add a `kustomization.yaml` at the cluster root.** Rejected because `flux bootstrap` owns the root sweep and the current auto-sweep model already solves discovery safely.
