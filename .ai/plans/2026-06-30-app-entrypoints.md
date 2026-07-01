# Per-App Flux Entrypoints Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split Flux apps into independent per-app reconciliation units without changing Talos or core infrastructure.

**Architecture:** Keep `flux/clusters/kube1/` as the Flux auto-swept entrypoint directory and replace the single aggregate `apps.yaml` with one Flux `Kustomization` CR per app. Each app lives in its own `flux/apps/<app>/` directory and follows a Flux-native overlay chain, so app customization stays in the Flux tree and not in Ansible.

**Tech Stack:** Flux, Kustomize, YAML, GitOps.

## Global Constraints

- Do not change Talos config or the current core infrastructure split.
- `flux/clusters/kube1/` stays auto-swept by Flux and must not gain a `kustomization.yaml`.
- App resources are hand-authored in the Flux tree; Ansible does not generate app manifests.
- Each app gets its own `flux/apps/<app>/` directory and its own flat cluster-root Flux `Kustomization` CR at `flux/clusters/kube1/<app>.yaml`.
- Keep app customization Flux-native and repo-owned.

---

### Task 1: Remove the aggregate apps scaffold

**Files:**
- Delete: `flux/clusters/kube1/apps.yaml`
- Delete: `flux/apps/kustomization.yaml`
- Delete: `flux/apps/base/kustomization.yaml`
- Delete: `flux/apps/overlays/common/kustomization.yaml`
- Delete: `flux/apps/overlays/clusters/kube1/kustomization.yaml`

**Interfaces:**
- Consumes: existing empty aggregate apps chain.
- Produces: no aggregate apps kustomization root remains.

- [ ] **Step 1: Delete the aggregate Flux Kustomization CR**

Remove `flux/clusters/kube1/apps.yaml` so the cluster root no longer manages a single shared apps reconciliation unit.

- [ ] **Step 2: Delete the aggregate `flux/apps` kustomize root**

Remove the empty root shim and overlay chain under `flux/apps/`.

- [ ] **Step 3: Verify the deleted files are empty scaffolding**

Run: `kustomize build flux/apps`

Expected: failure because the aggregate root no longer exists, which is the intended end state.

### Task 2: Add the per-app Flux layout convention

**Files:**
- Create: `flux/apps/<app>/base/kustomization.yaml`
- Create: `flux/apps/<app>/overlays/common/kustomization.yaml`
- Create: `flux/apps/<app>/overlays/clusters/kube1/kustomization.yaml`
- Create: `flux/apps/<app>/kustomization.yaml`

**Interfaces:**
- Consumes: Flux-native per-app ownership model.
- Produces: a self-contained app directory that can be pointed at by a flat cluster-root Flux `Kustomization` CR.

- [ ] **Step 1: Scaffold the app base**

Create `flux/apps/<app>/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources: []
```

- [ ] **Step 2: Add the repo-wide app overlay**

Create `flux/apps/<app>/overlays/common/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
```

- [ ] **Step 3: Add the per-cluster app overlay**

Create `flux/apps/<app>/overlays/clusters/kube1/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../common
```

- [ ] **Step 4: Add the app root shim**

Create `flux/apps/<app>/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - overlays/clusters/kube1
```

### Task 3: Add the flat cluster-root Flux CR for the app

**Files:**
- Create: `flux/clusters/kube1/<app>.yaml`

**Interfaces:**
- Consumes: `flux/apps/<app>/kustomization.yaml`.
- Produces: one Flux `Kustomization` CR per app.

- [ ] **Step 1: Create the app entrypoint CR**

Create `flux/clusters/kube1/<app>.yaml`:

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

- [ ] **Step 2: Keep app reconciliation independent**

Do not reintroduce a single aggregate `apps` CR. Each app remains independently reconcilable and independently prunable.

### Task 4: Update cluster documentation

**Files:**
- Modify: `flux/clusters/kube1/README.md`
- Create: `flux/apps/README.md`

**Interfaces:**
- Consumes: the per-app layout and flat cluster-root CR convention.
- Produces: repo docs that explain how to add and own apps.

- [ ] **Step 1: Update the cluster entrypoint README**

Replace the `apps.yaml` description with the per-app CR convention. Make it clear that each app gets its own flat CR in `flux/clusters/kube1/` and that the directory stays auto-swept.

- [ ] **Step 2: Document the per-app Flux layout**

Add `flux/apps/README.md` documenting the `base/`, `overlays/common/`, `overlays/clusters/kube1/`, and root shim structure.

### Task 5: Verify the refactor

**Files:**
- No new files

**Interfaces:**
- Consumes: the new per-app directory and cluster-root CR layout.
- Produces: a validated repo state with the aggregate apps scaffold removed.

- [ ] **Step 1: Verify app kustomize roots build**

Run: `kustomize build flux/apps/<app>`

Expected: succeeds for the per-app root shim.

- [ ] **Step 2: Verify the cluster entrypoint remains valid**

Run: `kustomize build flux/clusters/kube1`

Expected: still fails for the cluster root directory itself, because it is a Flux entrypoint and not a kustomize project.

- [ ] **Step 3: Verify render drift stays clean**

Run: `ansible-playbook infra/ansible/playbooks/render-config.yml -e config_render_check=true`

Expected: no drift, because apps are not Ansible-rendered.

- [ ] **Step 4: Review the git diff**

Run: `git diff -- flux/apps flux/clusters/kube1`

Expected: only the app scaffold, the new app CR, and doc updates are present.
