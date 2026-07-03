# flux/apps

Per-app Flux reconciliation units. Each application gets its own `flux/apps/<app>/` directory and its own flat Flux `Kustomization` CR at `flux/clusters/<cluster>/apps/<app>.yaml`. Apps reconcile independently — there is no aggregate `apps` root.

## Per-app directory layout

```
flux/apps/<app>/
├── kustomization.yaml          # Root shim → overlays/clusters/<cluster>
├── base/
│   └── kustomization.yaml      # Shared base resources (e.g. HelmRelease manifests)
└── overlays/
    ├── common/
    │   └── kustomization.yaml  # Repo-wide defaults → ../../base
    └── clusters/
        └── example/
            └── kustomization.yaml  # Per-cluster overrides → ../../common
```

This follows the four-tier overlay ownership model from ADR-017:

| Layer | Path | Purpose |
|---|---|---|
| Base | `base/` | Template catalog — shared workload manifests |
| Common overlay | `overlays/common/` | Repo-wide preferences applied to all clusters |
| Per-cluster overlay | `overlays/clusters/<cluster>/` | Cluster-specific overrides |
| Root shim | `kustomization.yaml` | Entrypoint the Flux CR points at |

## Adding a new app

1. Scaffold the app directory:

   ```bash
   mkdir -p flux/apps/<app>/{base,overlays/{common,clusters/<cluster>}}
   ```

2. Create the four `kustomization.yaml` files following the chain above.

3. Create the cluster entrypoint CR at `flux/clusters/<cluster>/apps/<app>.yaml`:

   ```yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1
   kind: Kustomization
   metadata:
     name: <app>
     namespace: flux-system
   spec:
     dependsOn:
       - name: infrastructure-configs
     interval: 10m
     retryInterval: 1m
     timeout: 5m
     sourceRef:
       kind: GitRepository
       name: flux-system
     path: ./flux/apps/<app>
     prune: true
   ```

4. Add workload manifests (e.g. `HelmRelease`, `Deployment`, `Service`) under `base/`.

The per-app directory `flux/clusters/<cluster>/apps/` is auto-swept recursively by Flux — placing the CR there is sufficient for Flux to discover and reconcile it. No `kustomization.yaml` is needed at that level.

## Why per-app instead of aggregate

- **Independent reconciliation** — each app has its own interval, timeout, and health scope.
- **Independent pruning** — deleting an app's CR and directory removes only that app's resources.
- **Independent failure domain** — one broken app does not block all other apps.
- **Clear ownership** — each app is a self-contained directory with a single cluster entrypoint CR.
