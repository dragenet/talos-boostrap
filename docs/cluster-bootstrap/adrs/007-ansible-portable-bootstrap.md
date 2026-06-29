# ADR-007: Ansible as a portable, provider-agnostic cluster bootstrap layer

**Status:** Accepted

## Context

Bringing up a Talos cluster involves: resolving a Talos image schematic, provisioning VMs (if on a cloud), applying machine configs, bootstrapping etcd, and installing pre-Flux components (Cilium, CCM). These steps span provider-specific API calls and provider-agnostic Talos/Kubernetes operations. The bootstrap tooling should be reusable across providers without forking.

## Decision

Use **Ansible**, with one uniform rule across every tree: **provider-agnostic at the top, provider-specific under `providers/<name>/`.**

```
inventories/  common/   providers/hcloud/   providers/manual/
playbooks/    *.yml      providers/hcloud/*.yml
roles/        talos_*/   providers/hcloud/*/
config/       defaults/  overrides/          (+ providers/<name>.yaml)
flux/infrastructure/controllers/  _components/  generated/selected/  overlays/common/  overlays/clusters/<cluster>/
```

Provider is selected with `-i inventories/common -i inventories/providers/<name>`; provider-specific playbooks live at `playbooks/providers/<name>/`. All other playbooks are provider-agnostic.

The Ansible code is **cluster-template code**, not kube1-specific — it is the renderer/bootstrapper of the config compiler (ADR-011), reading committed `config/` and emitting Talos + Flux artifacts. It is planned to move to a dedicated template repo; cluster-specific values stay in `config/overrides/`.

## Consequences

- A new provider requires only a new `providers/<name>/` directory **in each tree** (inventories, playbooks, roles, flux, config) — the adding-a-provider contract is now a single uniform rule. No changes to provider-agnostic paths. Documented in `docs/adding-a-provider.md`.
- `manual` (bare-metal) is modelled as the "null provider" — it sits under `providers/` for symmetry but has the smallest surface (no CCM/CSI, no `--cloud-provider=external`, no dynamic inventory).
- Roles are idempotent and FQCN-only (`ansible.builtin.*`, `hetzner.hcloud.*`). A second run must be a no-op.
- Internal role var conventions (flat, namespaced `cluster_*`/`hcloud_*`/`talos_*`) are preserved: the compiler renders flat `group_vars` for roles to consume — `cluster.yaml`'s nested shape is the human surface only (ADR-011).

## Addendum — 2026-06-30

**Summary:** The cluster-specific configuration path referenced in the decision text has moved from the legacy `config/overrides/` tree to a cluster-scoped directory.

**What changed:**
- The statement that "cluster-specific values stay in `config/overrides/`" is outdated.
- Cluster-specific values now live under `config/clusters/<cluster>/` (e.g. `config/clusters/kube1/`).
- The Ansible bootstrap layer still reads committed `config/` and emits Talos + Flux artifacts; only the user-owned config path changed.
