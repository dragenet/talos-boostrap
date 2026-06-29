# ADR-011: Config compiler — single input, deterministic render, one writer per file

**Status:** Accepted

## Context

The template repo is cloned and customized per cluster. Customization must be:
- driven from a small, discoverable surface (not scattered across group_vars + Talos patches + Flux manifests);
- **conflict-free on template update** — pulling upstream improvements must not collide with a user's choices;
- GitOps-consistent — git is the source of truth, the cluster reconciles from it.

## Decision

A **config compiler**: a single declarative input is rendered deterministically into the Talos and Flux artifacts the cluster consumes.

### Inputs — two-layer overlay
```
config/
  defaults/      # TEMPLATE-OWNED — upstream updates land here; users never edit
  overrides/     # USER-OWNED — only what differs; template never ships/edits these
```
Render computes `deep_merge(defaults, overrides)`, overrides winning. Merge is **per-key**:
- **merge** — maps (`features`);
- **replace** — user-owned lists (`nodes`, `storage.volumes`);
- **compute** — unions (`talos.extensions` = base ∪ feature-implied ∪ `talos.extraExtensions`). The render **rejects** a direct override of a compute-key; users add via the `*Extra*` key.

### Render (`configure.yml`) emits
- flat `group_vars` (so existing role var conventions are untouched — see ADR-007);
- Talos machine-config patches (see ADR-013);
- the Flux `base/` kustomization listing enabled components (see ADR-012);
- the `cluster-vars` ConfigMap consumed by Flux `postBuild.substituteFrom`.

### One writer per file (the core invariant)
Every file is owned by exactly one of **template** / **render** / **user**; the other two never touch it.
- template: `config/defaults`, `_components/` bases, render logic, playbooks/roles;
- render: `group_vars`, Talos patches, Flux `base/`, `cluster-vars` — marked *generated, do not edit*;
- user: `config/overrides`, the Flux overlay + `patches/`, `nodes.yaml`, apps, SOPS secrets.

Rendered output is committed but is **not** a source of truth — CI asserts `render(inputs) == committed tree` to catch hand-edits and drift.

### Customization ladder
1. **Use as-is** — enable in `cluster.yaml`, render.
2. **Tweak** — keep enabled, add a kustomize patch in the user overlay (git only, Flux reconciles).
3. **Replace** — disable the feature in `cluster.yaml` (render drops its base), deploy your own version via Flux (git only).

## Consequences

- **Render sits outside Flux's reconcile loop.** This is a conscious trade: Talos *must* be Ansible-rendered regardless, so a local/CI render step is unavoidable; CI is the drift guard. The cluster still reconciles git declaratively via Flux.
- **Ansible/render is needed for:** feature enable/disable, and any Talos-machine-config change (CNI, disk/volumes, network, podCIDR, image extensions incl. Tailscale, kubelet args). **Pure-git, no Ansible:** patching enabled features, apps, Flux-only scalars, SOPS secret rotation.
- Template updates touch only template-owned files, so `git pull upstream` is conflict-free against user-owned config.
- Build the compiler in stages — MVP first (direct `cluster.yaml` + `substituteFrom` + hand-written component lists), then the overlay, validation, and patch wiring.

## Addendum — 2026-06-30

**Summary:** The two-layer input model has been renamed and restructured. The user-owned layer is now cluster-scoped under `config/clusters/<cluster>/` rather than the generic `config/overrides/` tree.

**What changed:**
- The `config/defaults/` + `config/overrides/` two-layer overlay model is now `config/defaults/` + `config/clusters/<cluster>/`.
- User-owned cluster configuration — including `cluster.yaml`, `nodes.yaml`, and Talos overlay manifests — lives under `config/clusters/<cluster>/` (e.g. `config/clusters/kube1/`).
- The `defaults/` + `clusters/<cluster>/` merge semantics described in this ADR remain unchanged; only the path and naming of the user-owned layer changed.
