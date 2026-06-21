# ADR-006: FluxCD for GitOps

**Status:** Accepted (bootstrap stub exists; not yet deployed)

## Context

All cluster resources beyond the initial CNI and CCM (which must run pre-Flux) should be managed declaratively from git, with automated reconciliation and drift detection.

## Decision

Use **FluxCD** as the GitOps engine. Flux watches a git repository and reconciles `Kustomization` and `HelmRelease` objects. The cluster entrypoint is `flux/clusters/<cluster-name>/`, which references `infrastructure` and `apps` kustomizations with explicit `dependsOn` ordering:

```
infrastructure/controllers → infrastructure/configs → apps
```

All Flux `Kustomization` resources set `prune: true` and `wait: true` (on infra layers). Helm chart and image versions are pinned — no `:latest` or floating tags.

## Consequences

- **Cilium and CCM are pre-Flux** — they must be installed before Flux bootstraps, because Cilium is a CNI dependency (pods can't schedule without it) and CCM must clear the `uninitialized` taint. `bootstrap-flux.yml` installs both via Helm before running `flux bootstrap`.
- Chose FluxCD over ArgoCD (heavier, UI-first, more complex RBAC model for a single-cluster homelab) and manual Helm (no reconciliation, drift goes undetected).
