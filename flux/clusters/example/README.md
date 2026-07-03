# flux/clusters/example

Flux cluster entrypoint for the `example` cluster — the directory `flux bootstrap` reconciles into `flux-system`. Owns the `infrastructure/` Kustomization roots and per-app Flux `Kustomization` CRs for this cluster. See [ADR-006](../../docs/adr/0006-fluxcd-gitops.md), [ADR-017](../../docs/adr/0017-shared-overlay-ownership.md), and [ADR-018](../../docs/adr/0018-flux-entrypoint-subdirectories.md).

## Files

| Path | Owner | Purpose |
|---|---|---|
| `infrastructure/core.yaml` | user | Two Flux `Kustomization` CRs: `infrastructure-controllers` and `infrastructure-configs`. `infrastructure-configs.dependsOn: [infrastructure-controllers]`. |
| `infrastructure/<name>.yaml` | user | Additional standalone Flux `Kustomization` CRs that are not part of the controllers/configs chain (e.g. cluster-specific infrastructure such as DNS). Each file is a single `Kustomization` and may declare its own `dependsOn` (typically `[infrastructure-configs]`). |
| `apps/<app>.yaml` | user | Per-app Flux `Kustomization` CR — one flat file per application. Each `dependsOn: [infrastructure-configs]` and points `path` at `./flux/apps/<app>`. See [flux/apps/README.md](../../apps/README.md) for the per-app layout convention. |
| `flux-system/` | bootstrap | Created by `flux bootstrap`. Bootstrap-owned — do not hand-edit. Does not exist pre-bootstrap. |

## Reconciliation order

```
infrastructure-controllers  →  infrastructure-configs  →  apps/<app>
```

The `infrastructure/<name>.yaml` files are additional standalone roots; each declares its own `dependsOn` chain. All Kustomizations set `prune: true`; infra layers set `wait: true` so dependents block until Ready (ADR-006, ADR-018).

## Why there is no `kustomization.yaml` here

This directory is a **Flux entrypoint**, not a kustomize project. The `infrastructure/` subdirectory, the `apps/` subdirectory, and the cluster root itself are all auto-swept recursively by the `flux-system` `Kustomization` (rooted at `path: ./flux/clusters/<cluster-name>` in `flux-system/gotk-sync.yaml`). The files inside are Flux `Kustomization` custom resources, not kustomize manifests — no `kustomization.yaml` is needed at any of these levels (ADR-006, ADR-018).

Consequently `kustomize build flux/clusters/example` fails locally. That is intentional. The composable four-tier overlay structure (template catalog → generated-selected → common overlay → per-cluster overlay, ADR-017) lives one level down at `flux/infrastructure/controllers/`, `flux/infrastructure/configs/`, and `flux/apps/<app>/` — `kustomize build` is meaningful there, not here.

Adding a `kustomization.yaml` pre-bootstrap would force a post-bootstrap amendment to list `flux-system/` (which `flux bootstrap` creates mid-flight). The window between "unlisted" and "listed" state is an operational risk — auto-sweep avoids it entirely.

## Ownership

**PER-CLUSTER USER-OWNED** (ADR-017) — except `flux-system/`, which is bootstrap-owned. Hand-edit only files under `infrastructure/` and `apps/`.
