# flux/clusters/example

Flux cluster entrypoint for the `example` cluster — the directory `flux bootstrap` reconciles into `flux-system`. Owns the `infrastructure/` Kustomization roots and per-app Flux `Kustomization` CRs for this cluster. See [ADR-006](../../../docs/cluster-bootstrap/adrs/006-fluxcd-gitops.md) and [ADR-017](../../../docs/cluster-bootstrap/adrs/017-shared-overlay-ownership.md).

## Files

| File | Owner | Purpose |
|---|---|---|
| `infrastructure.yaml` | user | Two Flux `Kustomization` CRs: `infrastructure-controllers` and `infrastructure-configs`. `infrastructure-configs.dependsOn: [infrastructure-controllers]`. |
| `<app>.yaml` | user | Per-app Flux `Kustomization` CR — one flat file per application. Each `dependsOn: [infrastructure-configs]` and points `path` at `./flux/apps/<app>`. See [flux/apps/README.md](../../apps/README.md) for the per-app layout convention. |
| `flux-system/` | bootstrap | Created by `flux bootstrap`. Bootstrap-owned — do not hand-edit. Does not exist pre-bootstrap. |

## Reconciliation order

```
infrastructure-controllers  →  infrastructure-configs  →  <app>
```

All Kustomizations set `prune: true`; infra layers set `wait: true` so dependents block until Ready (ADR-006).

## Why there is no `kustomization.yaml` here

This directory is a **Flux entrypoint**, not a kustomize project. `infrastructure.yaml` and per-app `<app>.yaml` files are Flux `Kustomization` custom resources, not kustomize manifests — kustomize-controller auto-sweeps the directory, so no `kustomization.yaml` is needed.

Consequently `kustomize build flux/clusters/example` fails locally. That is intentional. The composable four-tier overlay structure (template catalog → generated-selected → common overlay → per-cluster overlay, ADR-017) lives one level down at `flux/infrastructure/controllers/`, `flux/infrastructure/configs/`, and `flux/apps/<app>/` — `kustomize build` is meaningful there, not here.

Adding a `kustomization.yaml` pre-bootstrap would force a post-bootstrap amendment to list `flux-system/` (which `flux bootstrap` creates mid-flight). The window between "unlisted" and "listed" state is an operational risk — auto-sweep avoids it entirely.

## Ownership

**PER-CLUSTER USER-OWNED** (ADR-017) — except `flux-system/`, which is bootstrap-owned. Hand-edit only `infrastructure.yaml` and per-app `<app>.yaml` files.
