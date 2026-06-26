# flux/clusters/kube1

Flux cluster entrypoint for `kube1` — the directory `flux bootstrap` reconciles into `flux-system`. Owns the two `infrastructure/` and the `apps/` kustomization roots for this cluster. See [ADR-006](../../../docs/cluster-bootstrap/adrs/006-fluxcd-gitops.md) and [ADR-017](../../../docs/cluster-bootstrap/adrs/017-shared-overlay-ownership.md).

## Files

| File | Owner | Purpose |
|---|---|---|
| `infrastructure.yaml` | user | Two Flux `Kustomization` CRs: `infrastructure-controllers` and `infrastructure-configs`. `infrastructure-configs.dependsOn: [infrastructure-controllers]`. |
| `apps.yaml` | user | One Flux `Kustomization` CR: `apps`. `dependsOn: [infrastructure-configs]`. |
| `flux-system/` | bootstrap | Created by `flux bootstrap`. Bootstrap-owned — do not hand-edit. Does not exist pre-bootstrap. |

## Reconciliation order

```
infrastructure-controllers  →  infrastructure-configs  →  apps
```

All Kustomizations set `prune: true`; infra layers set `wait: true` so dependents block until Ready (ADR-006).

## Why there is no `kustomization.yaml` here

This directory is a **Flux entrypoint**, not a kustomize project. `infrastructure.yaml` and `apps.yaml` are Flux `Kustomization` custom resources, not kustomize manifests — kustomize-controller auto-sweeps the directory, so no `kustomization.yaml` is needed.

Consequently `kustomize build flux/clusters/kube1` fails locally. That is intentional. The composable four-tier overlay structure (template catalog → generated-selected → common overlay → per-cluster overlay, ADR-017) lives one level down at `flux/infrastructure/controllers/`, `flux/infrastructure/configs/`, and `flux/apps/` — `kustomize build` is meaningful there, not here.

Adding a `kustomization.yaml` pre-bootstrap would force a post-bootstrap amendment to list `flux-system/` (which `flux bootstrap` creates mid-flight). The window between "unlisted" and "listed" state is an operational risk — auto-sweep avoids it entirely.

## Ownership

**PER-CLUSTER USER-OWNED** (ADR-017) — except `flux-system/`, which is bootstrap-owned. Hand-edit only `infrastructure.yaml` and `apps.yaml`.
