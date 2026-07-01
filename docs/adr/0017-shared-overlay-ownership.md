---
title: "ADR-017: Shared overlay ownership model across Talos and Flux"
description: A three-tier overlay model (template base, user-global, per-cluster) applies to both Talos and Flux, each keeping its natural filesystem root.
type: adr
audience: [contributor, ai]
tags: [adr, talos, flux, architecture]
status: accepted
created: 2026-07-01
updated: 2026-07-01
---

# ADR-017: Shared overlay ownership model across Talos and Flux

**Status:** Accepted

## Context

ADR-016 established a new boundary for Talos configuration: human-authored Talos node configuration should be Talos-shaped machine configuration patch manifests, with shared base and reusable component fragments living outside any one cluster, and cluster-specific overlays living under `config/clusters/<cluster>/talos/`. Ansible remains the compiler/orchestrator, selecting components and composing the ordered patch stack.

ADR-010, ADR-011, and ADR-012 already document the current Flux approach: a catalog of template-owned component bases under `flux/infrastructure/controllers/_components/`, a render-owned `base/` kustomization that selects enabled components, and a user-owned overlay with `patches/` and a top-level `kustomization.yaml`. The cluster entrypoint lives under `flux/clusters/<cluster>/`.

These two domains had superficially different layouts. Talos inputs were moving toward `config/talos/base/`, `config/talos/components/`, and `config/clusters/<cluster>/talos/`, while Flux inputs remained under the self-contained `flux/` directory. The question was whether to impose the same physical root on both domains, or whether the ownership model could be shared while each domain kept its natural filesystem root.

Official Flux documentation recommends a self-contained repo structure with `apps/`, `infrastructure/`, and `clusters/<cluster>/`; cluster directories are bootstrap entrypoints; `apps/` and `infrastructure/` separation enables ordered reconciliation; base/overlay patterns are common; and `flux bootstrap` writes into `clusters/<cluster>/flux-system/`. Moving Flux under `config/` would fight that convention.

## Decision

1. kube1 uses a **three-tier overlay ownership model** for cluster inputs that are composed from shared defaults:
   - **template/shared base or component defaults** — reusable, cluster-agnostic defaults owned by the template;
   - **user-global overlay** — preferences that apply to every cluster owned by this user/repo;
   - **per-cluster overlay** — final overrides for a specific cluster instance such as `kube1`.
2. Later layers override earlier layers. The **per-cluster overlay is the final user-owned layer** for that cluster.
3. This model applies to **both Talos and Flux**, but each domain keeps its natural root:
   - **Talos:** shared inputs under `config/talos/...` plus cluster-specific inputs under `config/clusters/<cluster>/talos/...` (ADR-016).
   - **Flux:** self-contained under `flux/...`, with cluster entrypoints under `flux/clusters/<cluster>/...` (aligned with Flux docs).
4. **Flux must not be moved under `config/`.** It stays self-contained and aligned with the Flux docs' `apps/`, `infrastructure/`, `clusters/<cluster>` mental model.
5. Flux should move toward a docs-aligned structure in which shared bases/components and overlays are explicit. The illustrative target layout is:

   ```
   flux/
     infrastructure/
       controllers/
         components/              # shared reusable component catalog (current _components equivalent)
         base/                    # shared or render-selected base; exact role to be resolved during migration
         overlays/
           user/                  # user-global overlay/patches applied to all clusters
           clusters/
             kube1/               # per-cluster overlay/patches
       configs/
         base/
         overlays/
           user/
           clusters/
             kube1/
     apps/
       base/
       overlays/
         user/
         clusters/
           kube1/
     clusters/
       kube1/
         flux-system/             # Flux bootstrap-owned
         infrastructure.yaml
         apps.yaml
   ```

6. Talos should use the corresponding ownership model from ADR-016, illustrated as:

   ```
   config/
     talos/
       base/
       components/
       overlays/
         user/
     clusters/
       kube1/
         talos/
   ```

   These layouts are illustrative of ownership, not exact implementation prescriptions.

7. The exact naming of Flux `components/` versus the current `_components/`, and whether `base/` remains render-owned or becomes a shared human-authored base, are **implementation/migration questions**. This ADR records the desired ownership model and notes these as open migration concerns; it does not silently decide every filesystem rename.
8. Ansible/render may still generate selected Flux bases where feature enablement is structural — Kustomize has no conditionals — but the ownership of generated vs. user-global vs. cluster overlay must be explicit.
9. This ADR **refines** ADR-012 and ADR-016; it does not supersede them. It also references ADR-010 and ADR-011.

## Consequences

- **Pros**
  - One mental model across Talos and Flux: template defaults, then user-global preferences, then per-cluster differences.
  - Keeps Flux aligned with official Flux docs and community conventions.
  - Supports multiple clusters from the same repo without forking component manifests.
  - Lets users express repo-wide preferences without patching each cluster individually.
  - Makes per-cluster differences explicit and discoverable.
  - Reduces confusion between template-owned, render-owned, and user-owned files.

- **Cons**
  - Extra directory depth in both `config/` and `flux/`.
  - Render logic must compose and validate more layers.
  - Conflicts between user-global and per-cluster overlays must be understandable and well-documented.
  - Migration is required from the current Flux `_components/base/patches` layout.
  - There is a naming tension around `base/`: in the current Flux tree, `base/` is the render-owned selected set, while Flux docs and Kustomize conventions often use `base/` as a shared human-authored base. The migration must resolve this ambiguity explicitly.

- **Operational safety**
  - `flux/clusters/<cluster>/flux-system/` remains owned by `flux bootstrap`; do not hand-edit generated/bootstrap-owned files casually.
  - GitOps reconciliation ordering remains required: `infrastructure` before `apps`, and controllers before configs.
  - Rendered/generated files must continue to be marked "generated, do not edit" per ADR-011.

## Rejected alternatives

- **Move Flux under `config/`.** Rejected because it breaks the self-contained Flux repo structure recommended by upstream docs and bootstrap tooling.
- **Only per-cluster overlays with no user-global layer.** Rejected because it forces users to repeat repo-wide preferences in every cluster.
- **Keep the current Flux top-level `patches/` as the only user overlay.** Rejected because it collapses user-global and per-cluster concerns and does not scale to multiple clusters.
- **Make every cluster fork full component manifests.** Rejected because it abandons reuse and makes template updates expensive.
- **Make Talos and Flux use identical physical directory roots.** Rejected because each domain has different upstream conventions and tooling expectations; the model is conceptual and ownership-based, not a forced mirror.

## Relation to prior decisions

- **Refines ADR-016** by making the Talos overlay model one half of a cross-domain convention, and by adding an explicit user-global overlay layer on top of the shared base/components and per-cluster overlays.
- **Refines ADR-012** by keeping the catalog concept, but moving Flux toward a three-tier overlay structure (template defaults → user-global overlay → per-cluster overlay) instead of the current render-owned `base/` plus single user `patches/` overlay.
- **Builds on ADR-011**: the config compiler still renders structural selections, but must now compose across more explicitly named ownership layers.
- **Builds on ADR-010**: the Ansible-to-Flux handoff and pre-Flux layer remain unchanged; only the organization of Flux-owned inputs is affected.
