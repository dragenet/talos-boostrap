---
title: GitOps with Flux
description: Day-2 operations with FluxCD — adding apps, enabling features, monitoring reconciliations, and troubleshooting.
type: guide
audience: [user, operator]
tags: [flux, gitops, kubernetes]
status: stable
created: 2026-07-01
updated: 2026-07-03
related:
  - getting-started.md
  - configure-your-cluster.md
  - ../concepts/gitops-model.md
  - ../adr/0006-fluxcd-gitops.md
  - ../adr/0012-feature-catalog.md
  - ../adr/0018-flux-entrypoint-subdirectories.md
---

# GitOps with Flux

After the initial bootstrap, all cluster state is managed declaratively from git using FluxCD. Committing a change to the `flux/` tree is the primary day-2 operation — Flux detects the change, reconciles it against the cluster, and reports status.

For a conceptual overview of how Flux manages the cluster, see [GitOps Model](../concepts/gitops-model.md).

---

## How Flux Watches the Repo

The cluster entrypoint is `flux/clusters/<cluster-name>/`. Its `flux-system` `Kustomization` (written by `flux bootstrap`) is rooted at `path: ./flux/clusters/<cluster-name>` and auto-sweeps the directory recursively. Inside the cluster root, reconciliation is organized into two subdirectories:

- `infrastructure/` — the `infrastructure/core.yaml` pair of `Kustomization` CRs (`infrastructure-controllers` + `infrastructure-configs`) plus any additional standalone `Kustomization` CRs (e.g. `infrastructure/<name>.yaml` for cluster-specific infrastructure such as DNS). See [ADR-018](../adr/0018-flux-entrypoint-subdirectories.md).
- `apps/` — one `<app>.yaml` `Kustomization` CR per application, each `dependsOn: [infrastructure-configs]` and pointing `path` at `./flux/apps/<app>`.

```
infrastructure-controllers  →  infrastructure-configs  →  apps/<app>
```

`infrastructure-controllers` reconciles everything under `flux/infrastructure/controllers/` — cert-manager, ingress controllers, OpenEBS, and other catalog components. `infrastructure-configs` depends on it and reconciles `flux/infrastructure/configs/`. Additional `infrastructure/<name>.yaml` files declare their own `dependsOn` chains (typically `[infrastructure-configs]`).

All kustomizations set `prune: true` — resources removed from git are removed from the cluster. Infrastructure layers set `wait: true` so dependent layers do not start reconciling until their dependencies are healthy.

> **Do not hand-edit `flux/clusters/<cluster-name>/flux-system/`.** That directory is owned by `flux bootstrap` and is regenerated on every re-bootstrap.

### Flux Tree Layout

```
flux/
  infrastructure/
    controllers/
      _components/             # template-owned feature catalog
      generated/               # render output — do not edit directly
      overlays/
        user/                  # repo-wide patches applied to all clusters
        clusters/
          <cluster-name>/      # per-cluster patches
    configs/
      base/
      overlays/
        user/
        clusters/
          <cluster-name>/
  apps/
    base/
    overlays/
      user/
      clusters/
        <cluster-name>/
  clusters/
    <cluster-name>/
      flux-system/             # owned by flux bootstrap — do not edit
      infrastructure/
        core.yaml              # infrastructure-controllers + infrastructure-configs
        <name>.yaml            # additional standalone infrastructure Kustomizations
      apps/
        <app>.yaml             # per-app Flux Kustomization CRs (one per app)
```

---

## The Reconcile Loop

The day-2 workflow is:

```
Edit flux/ in git  →  git push  →  Flux detects change  →  Flux applies to cluster
```

Flux polls its `GitRepository` source at a configured interval (default: `1h`). To apply a change immediately without waiting, force a reconcile — see [Forcing a Reconcile](#forcing-a-reconcile) below.

---

## Adding a New Application

Applications live under `flux/apps/`. The `flux/apps/` tree follows the same three-tier overlay model as infrastructure:

```
flux/apps/
  base/
  overlays/
    user/
    clusters/
      <cluster-name>/
```

To add a new application:

1. **Create the base manifests** under `flux/apps/base/<app-name>/`:

   ```yaml
   # flux/apps/base/<app-name>/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - namespace.yaml
     - helmrelease.yaml   # or deployment.yaml, etc.
   ```

2. **Wire it into the cluster** by adding a Flux `Kustomization` to your cluster overlay or a shared user overlay. For example, add a file to `flux/apps/overlays/clusters/<cluster-name>/`:

   ```yaml
   # flux/apps/overlays/clusters/<cluster-name>/<app-name>.yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1
   kind: Kustomization
   metadata:
     name: <app-name>
     namespace: flux-system
   spec:
     interval: 1h
     retryInterval: 1m
     timeout: 5m
     sourceRef:
       kind: GitRepository
       name: flux-system
     path: ./flux/apps/base/<app-name>
     prune: true
   ```

3. **Commit and push:**

   ```shell
   git add flux/apps/
   git commit -m "flux: add <app-name>"
   git push
   ```

Flux reconciles the new `Kustomization` and deploys the application.

---

## Enabling a Catalog Feature

Template-owned features (cert-manager, ingress controllers, OpenEBS, cloud CSI drivers) are catalog entries under `flux/infrastructure/controllers/_components/`. Enabling a feature is structural — the config compiler adds it to the generated kustomization at `flux/infrastructure/controllers/generated/selected/`.

To enable a feature:

1. Set the feature flag in `config/clusters/<cluster-name>/cluster.yaml`:

   ```yaml
   features:
     certManager: true
     ingress: envoy-gateway   # none | envoy-gateway | traefik
     openebs: true
   ```

2. Re-run the render:

   ```shell
   # From infra/ansible/
   ansible-playbook playbooks/render-config.yml -e config_cluster=<cluster-name>
   ```

3. Commit the generated output and push:

   ```shell
   git add -A && git commit -m "config: enable <feature> for <cluster-name>"
   git push
   ```

Flux reconciles `infrastructure-controllers` and deploys the feature.

> **Note:** Some features are Talos-coupled (e.g. OpenEBS requires a storage volume configured in `cluster.yaml`) and additionally require re-running `bootstrap-talos.yml`. See [Configure Your Cluster](configure-your-cluster.md) for details.

### Customizing a Feature

To patch a catalog feature without replacing it, add a Kustomize strategic-merge or JSON patch in the appropriate overlay:

- **Repo-wide patch** — `flux/infrastructure/controllers/overlays/user/patches/`
- **Per-cluster patch** — `flux/infrastructure/controllers/overlays/clusters/<cluster-name>/patches/`

These patches target the HelmRelease or raw manifests. This is a pure git change; no render is required.

---

## Monitoring Flux

### See all reconciliations

```shell
flux get all
```

Shows all `Kustomization`, `HelmRelease`, `GitRepository`, and other Flux resources with their status, last applied revision, and ready condition.

### See Flux controller logs

```shell
flux logs
```

Streams recent log lines from all Flux controllers in `flux-system`. Useful for diagnosing reconciliation errors.

To follow logs in real time:

```shell
flux logs --follow
```

### Forcing a Reconcile

To apply a commit immediately without waiting for the next interval:

```shell
# Force reconcile the GitRepository source first
flux reconcile source git flux-system

# Then reconcile a specific kustomization
flux reconcile kustomization infrastructure-controllers
flux reconcile kustomization infrastructure-configs
```

---

## Troubleshooting

### Kustomization stuck in "not ready"

```shell
flux describe kustomization <name>
```

Common causes:
- **Build failure** — a resource in the kustomization path has a syntax error or references a missing resource. Check `flux logs` for `kustomize build` errors.
- **Dependency not ready** — a `dependsOn` target has not reached the ready state. Check the dependency first.
- **Missing Secret or ConfigMap** — a `postBuild.substituteFrom` source is absent. Ensure any SOPS-encrypted secret files have been committed and decrypted correctly.

### HelmRelease upgrade failing

```shell
flux describe helmrelease <name>
```

Check the `Status.Conditions` and `Status.History` fields. Common causes:
- **Invalid values** — a Kustomize patch applies a value the chart does not accept. Check the patch against the chart's `values.yaml`.
- **Pre-upgrade hook failure** — check the pod logs for the hook job.
- **Chart unavailable** — the `HelmRepository` source may be stale. Force-reconcile it:
  ```shell
  flux reconcile source helm <repository-name>
  ```

### See what Flux would apply

```shell
flux diff kustomization <name>
```

Shows the diff between the current cluster state and what Flux would apply on the next reconcile.

---

## Day-2 Operations Reference

| Operation | How |
|---|---|
| Add an app | Add manifests under `flux/apps/`; commit and push |
| Enable a catalog feature | Set in `cluster.yaml`, run render, commit generated output, push |
| Patch a feature | Add patch in `overlays/clusters/<cluster-name>/patches/`; commit and push |
| Disable a feature | Unset in `cluster.yaml`, run render, commit and push |
| Upgrade a Helm chart | Update `spec.chart.spec.version` in the HelmRelease; commit and push |
| Rotate a SOPS secret | Re-encrypt the secret file; commit and push |
| Force reconcile | `flux reconcile kustomization <name>` |
| Talos machine-config change | Update `cluster.yaml`, run render, run `bootstrap-talos.yml` |

---

## Further Reading

- [GitOps Model](../concepts/gitops-model.md) — conceptual overview of Flux management, feature taxonomy, overlay ownership
- [Configure Your Cluster](configure-your-cluster.md) — changing cluster settings, enabling features, the render cycle
- [ADR-006: FluxCD for GitOps](../adr/0006-fluxcd-gitops.md)
- [ADR-012: Feature Catalog](../adr/0012-feature-catalog.md)
- [ADR-018: Flux entrypoint subdirectories](../adr/0018-flux-entrypoint-subdirectories.md)
