# ADR-016: Talos configuration boundary — Talos-shaped patches, Ansible as compiler

**Status:** Accepted

## Context

The repository had drifted toward a custom abstraction on top of Talos machine configuration. A concrete example surfaced immediately before this decision: the config model exposed a field such as `talos.network.defaultInterface.interface/deviceSelector`, and the schema typed `deviceSelector` as `string|null`. The template could render selector-like output, but the type was wrong; correcting it to `object|null` only fixed the immediate bug. It revealed the broader smell: kube1 was building an "almost Talos" schema instead of letting users write Talos-shaped configuration directly from the Talos documentation.

Talos already has a well documented, stable mechanism for this: machine config patch manifests applied via strategic-merge patching, including multi-document patch files. `talosctl gen config` accepts multiple ordered `--config-patch`, `--config-patch-control-plane`, and `--config-patch-worker` flags. The Talos project recommends keeping reproducible inputs (patch files, cluster name/endpoint/version contract) in Git, regenerating machine configs on demand, and not committing the generated machine configs.

The existing render flow already acts as a compiler/orchestrator: it loads defaults and overrides, validates them, renders generated Ansible group_vars, generated Talos patch outputs under `infra/talos/patches/generated/`, and the generated Flux controller base. That orchestrator role is valuable, but its responsibility should be reduced to composing Talos-shaped inputs rather than translating a bespoke schema into Talos.

This ADR refines and supersedes ADR-013, which introduced layered Talos config with structured helpers plus raw fragments. The new boundary keeps the layered concept but removes the broad parallel schema.

## Decision

1. kube1 must not define a broad parallel schema for Talos machine configuration.
2. Human-authored Talos node configuration must be stored as Talos-shaped machine config patch manifests, so users can follow Talos documentation directly.
3. Ansible remains responsible for orchestration and integration glue only: loading defaults and the selected cluster config, choosing provider/feature-specific patch components, composing patch order, rendering generated outputs, and injecting values that must be computed or enforced by kube1 integration.
4. Shared Talos base lives outside clusters, e.g. `config/talos/base/`, because `kube1` is a cluster instance/customization, not the base.
5. Reusable feature/provider Talos patch fragments live outside clusters, e.g. `config/talos/components/`, selected by Ansible based on provider/features. Examples: external cloud provider, Cilium requirements, future hcloud CSI/storage requirements.
6. Cluster-specific Talos overlays live under `config/clusters/<cluster>/talos/`, e.g. `config/clusters/kube1/talos/`, with optional per-role and per-node patch manifests.
7. Playbooks accept a cluster selector such as `-e config_cluster=kube1`, allowing future clusters to consume the shared base and components.
8. Generated machine configs and secrets remain uncommitted/gitignored. Generated integration patch outputs may continue to be rendered into `infra/talos/patches/generated/` or an equivalent generated output used by current roles, but the source-of-truth human Talos patches live under `config/talos/` and `config/clusters/<cluster>/talos/`.
9. Avoid the ambiguous term `talosconfig` for these files; use "Talos machine configuration patch manifests" or "Talos node configuration manifests".
10. The render/compiler path must guard against Talos merge hazards, especially list merge keys: do not split ownership of the same `machine.network.interfaces` item across layers unless they use the same merge key and the ordering/ownership is deliberate.

### Illustrative target structure

```
config/
  defaults/
    cluster.yaml                 # non-Talos kube1/provider/features defaults
  talos/
    base/                        # template-owned, cluster-agnostic Talos-shaped patch manifests
      all.yaml
      controlplane.yaml
      worker.yaml
    components/                  # reusable feature/provider Talos fragments selected by Ansible
      external-cloud-provider.yaml
      cilium.yaml
      hcloud-csi.yaml
  clusters/
    kube1/
      cluster.yaml               # kube1 provider/features/versions/Flux repo/etc.
      nodes.yaml                 # kube1 node list/roles/provider facts
      talos/                     # user-owned kube1 Talos-shaped overlays
        all.yaml
        controlplane.yaml
        worker.yaml
        nodes/
          kube1-hb1.yaml
          kube1-hb2.yaml
          kube1-hb3.yaml
```

This layout is illustrative, not an exact implementation prescription.

### Patch stack concept

The final node machine config is produced by layering Talos-shaped inputs in order:

```
upstream Talos generated machine config
+ config/talos/base/*
+ config/clusters/<cluster>/talos/*
+ config/talos/components/* selected by provider/features
+ Ansible-generated computed integration fragments, if needed
+ per-node overlays
= final node machine config rendered into gitignored secrets/
```

Ansible chooses which components to include (for example external cloud provider when `cloud_controller_manager.enabled` is true, or Cilium-specific kubelet/network settings when `cni.name: cilium`) and composes the ordered patch stack passed to `talosctl gen config`. It does not re-express those patches as its own schema.

## Consequences

- **Pros**
  - Aligns with Talos documentation and upstream conventions.
  - Reduces maintenance burden of a custom schema that must track every Talos field.
  - Improves user understanding because the inputs are real Talos patch manifests.
  - Avoids lossy abstractions where a narrowly typed custom field cannot express a valid Talos construct.
  - Supports future clusters by separating shared base/components from cluster-specific overlays.
  - Keeps Ansible as compiler/orchestrator rather than a second Talos API.

- **Cons**
  - Users must understand Talos patch syntax and strategic-merge semantics.
  - `render-config` must validate and compose ordered patch stacks carefully, especially around list merge keys.
  - Migration from the current `talos.*` custom fields is required.
  - Some Ansible-generated integration fragments still exist and must be clearly named and owned.

- **Operational safety**
  - Generated machine configs and Talos secrets remain gitignored; they are regenerated locally from committed patches plus the Talos secrets bundle.
  - Docs-before-code still applies: read Talos documentation when changing node configuration.
  - Machine config changes may require Talos validation and careful apply mode (`--mode=no-reboot` first, then a planned reboot/upgrade cycle if Talos reports a reboot is required).

## Rejected alternatives

- **Keep expanding the custom `talos.*` schema.** Rejected because it duplicates Talos documentation, creates lossy types, and increases maintenance every Talos release.
- **Put the shared base under `config/clusters/kube1/`.** Rejected because `kube1` is one cluster instance; the base should be reusable and cluster-agnostic.
- **Commit generated machine configs.** Rejected because they embed cluster CA and keys; Talos recommends regenerating them on demand from committed inputs.
- **Make every Talos value an Ansible variable.** Rejected because it recreates the parallel schema problem and turns Ansible into a second Talos API instead of an orchestrator.

## Relation to prior decisions

- Supersedes ADR-013, which introduced layered Talos config. The layered concept remains, but the boundary between structured helpers and raw fragments is redrawn: Talos-shaped patches are now the default, and custom schema is minimized.
- Builds on ADR-011 (config compiler): the compiler/orchestrator still loads defaults and cluster config, but now renders fewer generated artifacts and composes more Talos-native inputs.
- Interacts with ADR-012 (feature catalog): Talos-coupled features still need both a Flux component and a Talos component, but the Talos component is now a Talos-shaped patch fragment under `config/talos/components/` rather than a custom schema translation.
