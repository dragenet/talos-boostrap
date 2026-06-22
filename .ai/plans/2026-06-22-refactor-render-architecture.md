# 2026-06-22 — Refactor render/config-compiler architecture

## Goal and scope

Finish the kube1 refactor described by ADR-010 through ADR-013 by introducing the render/config-compiler architecture and migrating the current hand-authored ADR-007-era files into generated artifacts without behavior change.

This is **refactor work only**. It must preserve the current effective cluster configuration while creating the structure needed for later Cilium / hcloud CCM / hcloud CSI / Flux-bootstrap feature work. The existing feature plan at `.ai/plans/2026-06-22-cilium-ccm-csi.md` must be revised after this refactor lands.

Out of scope for this plan:

- Installing Cilium, hcloud CCM, hcloud CSI, Flux, cert-manager, OpenEBS, or ingress.
- Changing live cluster state.
- Creating billable Hetzner resources.
- Resolving the ADR-005 vs ADR-010 CCM ownership conflict silently.
- Replacing ADR-014 dynamic inventory, which is already built and should only be cross-referenced.

## Per-ADR build status

| ADR | Status for this plan | Notes |
|---|---:|---|
| ADR-010 Ansible-to-Flux provider handoff | **Not built** | Implement the render-owned Flux `base/`, user overlay, and provider-derived pre-Flux variable plumbing only. Do not install the features yet. |
| ADR-011 Config compiler | **Not built** | Implement `config/defaults` + `config/overrides`, deterministic render, generated headers, and one-writer-per-file checks. |
| ADR-012 Feature catalog | **Not built** | Create sparse catalog layout under `flux/infrastructure/controllers/_components/`, but leave actual feature content deferred unless needed as empty placeholders. |
| ADR-013 Layered Talos config | **Not built** | Migrate current Talos patches into layered structured-helper + raw-fragment model while preserving current output. |
| ADR-014 Node discovery via inventory | **Built** | Do not rework. The compiler must respect the existing hostvar contract: `node_public_ip`, `node_private_ip`, `node_role`, and role groups. |

## Current-state facts from repo context

These facts come from the current files and should be treated as the migration baseline, not as feature-complete state:

- `infra/ansible/inventories/common/group_vars/all/cluster.yml` currently owns cluster identity and API defaults: `cluster_name: kube1`, `cluster_vip: 10.0.0.10`, `cluster_endpoint: https://{{ cluster_vip }}:6443`, `kubernetes_version: v1.36.0`, empty Talos/Kubernetes cert SAN lists, role short-name mapping, and local Ansible connection settings.
- `infra/ansible/inventories/common/group_vars/all/talos.yml` currently owns `talos_version: v1.13.3`, `talos_extensions: [siderolabs/qemu-guest-agent]`, and Talos secrets/patches paths.
- `infra/ansible/inventories/common/group_vars/all/flux.yml` currently owns `flux_repo_url: ''`, `flux_repo_branch: master`, and `flux_path: ./flux/clusters/kube1`.
- `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.yml` currently owns hcloud provider settings: `nbg1`, server-type fallback lists, network/firewall names and CIDRs, open firewall mode, empty admin IPs, 3 hybrid nodes, and node IP offset 11.
- `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml` is the encrypted hcloud token source. It must remain user/secret-owned and must not be rendered or decrypted by this refactor.
- `infra/ansible/inventories/providers/manual/hosts.yml` is the built ADR-014 static inventory example. It has node hostvars and no provider group vars yet.
- `infra/talos/patches/common.yaml.j2` currently renders only `machine.install.image` plus optional Talos API `certSANs`.
- `infra/talos/patches/controlplane.yaml.j2` currently renders private-network VIP on `eth1`, `cluster.network.cni.name: none`, `cluster.proxy.disabled: true`, and optional kube-apiserver `certSANs`.
- `infra/ansible/roles/talos_config/tasks/main.yml` currently templates the two patch files into `infra/talos/secrets/`, runs `talosctl gen config` with one all-node patch and one control-plane patch, then applies per-node inline patches for hostname, Kubernetes node-role label, and hybrid scheduling.
- `flux/infrastructure/controllers/kustomization.yaml` and `flux/infrastructure/configs/kustomization.yaml` are empty `resources: []` placeholders.
- `flux/clusters/kube1/infrastructure.yaml` already has two Flux `Kustomization` resources with `infrastructure-configs` depending on `infrastructure-controllers`; both set `interval`, `retryInterval`, `timeout`, `prune: true`, and `wait: true`.
- `infra/ansible/roles/flux_bootstrap/tasks/main.yml` is still a stub. Do not fill in feature installs in this refactor.
- `infra/ansible/requirements.yml` pins `hetzner.hcloud` and `community.sops`; no render-specific dependency is currently declared.
- Graphify still references a deleted `.ai/plans/2026-06-21-config-compiler.md`. Treat it as stale context; this plan replaces it.

## Official docs that must be consulted before implementation

Implementors must re-check these official docs immediately before coding. The points below are planning constraints, not a substitute for final doc verification.

- Talos config patching: <https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/system-configuration/patching.md>
  - Multiple `--config-patch` flags are applied in order; later patches win.
  - A single patch file cannot modify the same document more than once, so generated Talos layers should be separate ordered files.
  - `network.interfaces` merge on `interface:` or `deviceSelector:`; `vlans` merge on `vlanId:`.
- Talos reproducible machine configuration: <https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/system-configuration/reproducible-machine-configuration.md>
  - Keep secrets separate, pin Talos version, and regenerate from committed patches rather than committing rendered machine configs.
- Talos disk management and `UserVolumeConfig`: <https://docs.siderolabs.com/talos/v1.13/configure-your-talos-cluster/storage-and-disk-management/disk-management/common.md> and <https://docs.siderolabs.com/talos/v1.13/reference/configuration/block/uservolumeconfig>
  - Storage selection uses `diskSelector`, not network `deviceSelector`.
  - `UserVolumeConfig` is formatted/mounted; OpenEBS LocalPV/LVM-style raw-device use must validate against `RawVolumeConfig` or whole-disk requirements when that feature is later implemented.
- Kustomize components guide/reference: <https://kubectl.docs.kubernetes.io/guides/config_management/components/> and <https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/components/>
  - Official Kustomize `kind: Component` entries are transform-only and cannot use `resources:` or nested `components:`.
  - Because controller catalog entries need to introduce HelmRepository/HelmRelease resources later, `_components/` should be treated as a repo catalog of ordinary kustomize bases unless an entry is genuinely only a transformer/generator component.

## Target ownership model and one-writer invariant

The refactor must make ownership visible in file paths and generated headers:

| Owner | Files/directories | Rule |
|---|---|---|
| Template | `config/defaults/`, `infra/ansible/playbooks/render-config.yml`, `infra/ansible/roles/config_render/`, `flux/infrastructure/controllers/_components/`, render templates | Updated by template/repo maintainers only. |
| User | `config/overrides/`, `config/overrides/nodes.yaml`, `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml`, manual `hosts.yml`, Flux overlay `kustomization.yaml`, `flux/infrastructure/controllers/patches/`, apps | Users edit these; render must never overwrite secrets or user patches. |
| Render | generated Ansible group_vars, generated Talos patch templates/fragments, `flux/infrastructure/controllers/base/`, generated `cluster-vars` ConfigMap | Mark with `# Generated by render-config; do not edit.` CI/review asserts committed output equals render output. |

Avoid mixed ownership inside a single file. If an existing file contains both user and generated concerns, migrate the user concern into `config/overrides/` first, then let render own the file.

## Cluster schema and input layout

Owner: `ansible-implementor`

Use `cluster.yaml` as the logical schema file under the two-layer config directory from ADR-011:

```text
config/
  defaults/
    cluster.yaml       # template-owned defaults and schema-shaped examples
  overrides/
    cluster.yaml       # user-owned kube1/provider choices copied from current group_vars
    nodes.yaml         # user-owned per-node hardware selectors and raw node patches
```

Initial schema proposal:

```yaml
cluster:
  name: kube1
  kubernetesVersion: v1.36.0
  vip: 10.0.0.10
  endpoint: "https://{{ cluster_vip }}:6443"
  certSANs:
    talos: []
    kubernetes: []
  nodeRoleShort:
    controlplane: cp
    hybrid: hb
    worker: wk

provider:
  name: hcloud      # hcloud | manual

flux:
  repoUrl: ""
  repoBranch: master
  path: ./flux/clusters/kube1

talos:
  version: v1.13.3
  extensions:
    base:
      - siderolabs/qemu-guest-agent
    extra: []       # user extension hook; renderer computes final talos_extensions
  install:
    # Keep optional in the initial migration. Current config only sets installer image.
    disk: null
    diskSelector: null
  network:
    vip: 10.0.0.10
    defaultInterface:
      # ADR-013 target is deviceSelector. For behavior-preserving migration,
      # render may keep interface: eth1 until a safe deviceSelector is supplied.
      interface: eth1
      deviceSelector: null
  patches:
    all: []
    controlplane: []

features:
  cni: cilium
  certManager: false
  ingress: none
  openebs: false
  hcloudCsi: false

storage:
  volumes: []

hcloud:
  location: nbg1
  serverTypes: [cx33, cx43]
  serverTypesPerRole: {}
  builderServerTypes: [cx23, cx33, cpx22]
  privateNetworkName: kube1
  privateNetworkCidr: 10.0.0.0/16
  privateSubnetCidr: 10.0.0.0/24
  firewallName: kube1
  sshKeyName: kube1-provisioning
  firewallOpenAccess: true
  adminIps: []
  topology:
    controlplane: 0
    hybrid: 3
    worker: 0
  serversIpOffset: 11
```

`config/overrides/nodes.yaml` schema proposal:

```yaml
nodes: []
# Example shape for future/manual hardware-specific config:
# - name: kube1-hb1
#   role: hybrid
#   network:
#     interfaces:
#       - name: private
#         deviceSelector: null
#   patch: []
```

Merge/validation rules:

- Maps deep-merge with overrides winning.
- User-owned lists such as `nodes` and `storage.volumes` replace defaults.
- Computed keys are rejected if directly overridden. Examples: final `talos_extensions`, rendered `bootstrap_pre_flux_components`, and Flux enabled-component lists.
- `talos.extensions.base ∪ feature-implied extensions ∪ talos.extensions.extra` computes generated `talos_extensions`.
- Provider-gated fields are validated: hcloud-only settings are rejected for `provider.name: manual`; hcloud feature toggles are rejected on manual.
- Feature constraints from ADR-012 are enforced even if catalog content is sparse: ingress pick-one, cert-dependent features require cert-manager, provider-gated features require matching provider, and user patches must target enabled components.

## Compiler implementation proposal

Owner: `ansible-implementor`

Implement the compiler using repo-native Ansible rather than introducing a separate language/toolchain:

```text
infra/ansible/playbooks/render-config.yml
infra/ansible/roles/config_render/
  defaults/main.yml
  tasks/main.yml
  tasks/load.yml
  tasks/validate.yml
  tasks/render_ansible.yml
  tasks/render_talos.yml
  tasks/render_flux.yml
  tasks/check.yml
  templates/
```

The render playbook should run on `localhost`, gather no facts, and be side-effect-free outside the repository tree. It must not contact Hetzner, Kubernetes, Talos APIs, or Flux.

Inputs:

- `config/defaults/cluster.yaml`
- `config/overrides/cluster.yaml`
- `config/overrides/nodes.yaml`
- Current committed user-owned Flux overlay/patch directory for validation only.

Outputs:

- Generated flat group vars preserving existing role variable names:
  - `infra/ansible/inventories/common/group_vars/all/cluster.yml`
  - `infra/ansible/inventories/common/group_vars/all/talos.yml`
  - `infra/ansible/inventories/common/group_vars/all/flux.yml`
  - `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.yml` for non-secret hcloud vars.
  - A new generated manual-provider group-vars file such as `infra/ansible/inventories/providers/manual/group_vars/all/bootstrap.yml` or `manual.yml` if needed for `bootstrap_pre_flux_components`.
- Generated Talos patch templates/fragments under a render-owned subdirectory, for example:
  - `infra/talos/patches/generated/all/00-install.yaml.j2`
  - `infra/talos/patches/generated/all/10-cert-sans.yaml.j2`
  - `infra/talos/patches/generated/controlplane/00-network.yaml.j2`
  - `infra/talos/patches/generated/controlplane/10-cni-proxy.yaml.j2`
  - `infra/talos/patches/generated/controlplane/90-user-raw.yaml.j2`
  - optional generated per-node patch files if `nodes.yaml` contains raw node patches.
- Flux render output:
  - `flux/infrastructure/controllers/base/kustomization.yaml`
  - generated `cluster-vars` ConfigMap, preferably under `flux/infrastructure/controllers/base/cluster-vars.yaml` or a separate generated config location consumed by the cluster entrypoint.

Generated file conventions:

- Put a generated header at the top of every generated YAML/Jinja file.
- Preserve current flat variable names so existing roles keep working during the refactor.
- Keep `hcloud.sops.yaml` out of all render tasks.
- Keep rendered Talos machine configs and talosconfig under `infra/talos/secrets/` gitignored as today.

Render modes:

- Default: write generated files.
- Check mode/tag: render to a temporary directory and compare against committed generated files for equivalence/drift. This supports CI/review without making edits.
- Migration mode can be a documented one-time sub-step, not a separate permanent workflow.

## Migration approach without data loss

Owner: `ansible-implementor`

1. Copy every current value from `cluster.yml`, `talos.yml`, `flux.yml`, and non-secret `hcloud.yml` into `config/overrides/cluster.yaml` using the new schema. Preserve comments where they explain operational risk, either in schema comments or docs.
2. Seed `config/defaults/cluster.yaml` with template defaults that match the accepted architecture but do not override kube1-specific choices.
3. Seed `config/overrides/nodes.yaml` with `nodes: []` initially, because ADR-014 inventory already supplies node public/private IPs and roles.
4. Introduce generated headers on output files only after the render can reproduce the current content.
5. Run the render into a temporary directory and compare generated outputs to the current hand-authored baseline.
6. Only then replace the hand-authored group_vars and Talos patches with generated equivalents.
7. Document that future edits go through `config/overrides/`, not the generated files.

Quality gate: rendered output must be behavior-preserving. The first accepted render may differ in comments and generated headers, but semantic YAML values and Talos patch meaning must match the current baseline. Any intentional semantic difference requires a separate explicit decision.

## Talos layered patch plan

Owner: `ansible-implementor`

Refactor `talos_config` to consume ordered patch lists instead of exactly two hard-coded patch templates.

Target generation/apply order:

1. Talos base from `talosctl gen config`.
2. Layer 1 structured helper patches from `cluster.yaml`:
   - install image placeholder (`talos_installer_image`) and optional install disk/diskSelector.
   - Talos API SANs.
   - control-plane/private API VIP network patch.
   - CNI disabled and kube-proxy disabled, preserving current Cilium-ready behavior.
   - storage volume patches only when later features enable them; do not invent OpenEBS output in this refactor.
3. Layer 2 raw fragments:
   - `talos.patches.all`.
   - `talos.patches.controlplane`.
   - `nodes.yaml` per-node `patch` entries.
4. Role-required per-node patches for hostname, node-role label, and hybrid scheduling, with ordering documented so user raw node patches cannot accidentally erase required identity unless explicitly allowed and reviewed.

Important constraints from Talos docs:

- Use multiple ordered patch files rather than combining repeated mutations of the same document into one file.
- Keep generated machine configs in `infra/talos/secrets/`; do not commit them.
- Support network `deviceSelector` in the schema and renderer. The user authorizes a live `talosctl` probe to discover the real hardware identifiers (see "Talos deviceSelector probe" section). Use the probe output to build a real `deviceSelector`; do not keep `interface: eth1` as an interim unless the probe proves `deviceSelector` is not viable for Hetzner CX33.
- Distinguish network `deviceSelector` from storage `diskSelector` in variable names and validation.

Behavior-preserving target for the initial render:

- Generated all-node Talos patch must still set `machine.install.image: {{ talos_installer_image }}` and optional Talos API SANs.
- Generated control-plane patch must still set the VIP on the private interface (using `deviceSelector` from the `talosctl` probe, or `interface: eth1` only if the probe proves `deviceSelector` is not viable), disable in-cluster CNI management with `cluster.network.cni.name: none`, disable kube-proxy, and preserve optional kube-apiserver SANs.
- No new `cluster.externalCloudProvider`, disk, volume, OpenEBS, or CCM settings should appear in this refactor unless explicitly confirmed as part of a later feature plan.

## Flux catalog/base/patches layout

Owner: `k8s-implementor`

Create the layout from ADR-010/012 without populating feature resources yet:

```text
flux/infrastructure/controllers/
  _components/          # template-owned sparse catalog
    README.md           # explains catalog contract and ordinary-base vs Kustomize Component distinction
    providers/
      hcloud/
        .gitkeep or README.md
  base/                 # render-owned selected components for this cluster
    kustomization.yaml
    cluster-vars.yaml   # if cluster-vars belongs at controllers level
  patches/              # user-owned patch directory
    README.md or .gitkeep
  kustomization.yaml    # user-owned overlay
```

Recommended `kustomization.yaml` ownership:

- `flux/infrastructure/controllers/kustomization.yaml` remains user-owned overlay and should reference `base/` plus user patches.
- `flux/infrastructure/controllers/base/kustomization.yaml` is render-owned and should list enabled catalog entries via `resources:`.
- `_components/<feature>/` entries that introduce HelmRepository/HelmRelease resources should be ordinary kustomize bases, not `kind: Component`, because official Kustomize components cannot contain `resources:`.
- If a future catalog entry is truly transform-only, it may use `kind: Component` and be listed under `components:`, but that is not the default for controllers.

Sparse catalog rule:

- This refactor may create directories and README/.gitkeep placeholders only.
- Do not add Cilium, CCM, CSI, cert-manager, OpenEBS, or ingress manifests here. Those are deferred feature work.

Patch validation rule:

- Implement a render validation hook that can later assert every file in `patches/` targets an enabled component. For the initial sparse catalog, document the rule and make validation pass for an empty patch directory.

## `bootstrap_pre_flux_components` wiring

Owner: `ansible-implementor`

Add/refactor provider group-vars output so the future `bootstrap-flux.yml` can iterate a provider-keyed list without feature code being implemented now.

Target generated values:

```yaml
# hcloud provider generated group vars
bootstrap_pre_flux_components:
  - cilium
  - hcloud-ccm
```

```yaml
# manual provider generated group vars
bootstrap_pre_flux_components:
  - cilium
```

Notes:

- This wiring follows ADR-010/ADR-005 bootstrap ordering but does not install anything yet.
- If the implementation prefers role-local naming such as `flux_bootstrap_pre_flux_components`, it must either preserve a compatibility alias or update the plan/docs consistently. The user explicitly requested `bootstrap_pre_flux_components`; prefer that exact name unless there is a strong repo-convention reason not to.
- For `provider.name: manual` and `features.cni: none` in the future, render should be able to compute an empty list. Do not implement feature behavior beyond the current cilium baseline unless required for validation.

## ADR-005 vs ADR-010 CCM ownership conflict — RESOLVED

Owner: `adr-writer` (after implementation, as a follow-up ADR update)

Conflict:

- ADR-005 says hcloud CCM runs as a plain Helm release outside Flux and is not reconciled by Flux.
- ADR-010 says hcloud CCM is installed pre-Flux and then adopted by Flux.

Decision (user-confirmed 2026-06-22): **ADR-010 supersedes ADR-005.** Higher ADR numbers win by repo convention. CCM installs pre-Flux (to clear the uninitialized taint), then Flux adopts the same Helm release for steady-state ownership.

Rationale:

- The bootstrap reason from ADR-005 remains true: CCM must run before Flux when kubelet uses external cloud provider, because it clears `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule`.
- Long-term ownership by Flux better matches ADR-011's GitOps-consistent model and reduces drift/upgrades outside Git.
- Adoption keeps the bootstrap layer minimal over time: Ansible installs only the pre-Flux release needed to unblock scheduling; Flux later reconciles the same release name/namespace/chart values.
- The trade-off is adoption sensitivity: release names, namespaces, chart versions, and values must match exactly to avoid uninstall/reinstall churn. That belongs in the later feature plan and reviewer checklist.

Follow-up for `adr-writer`: update ADR-005 with a supersession note pointing to ADR-010, and ensure `_components/providers/hcloud/` will model CCM as a Flux-adopted HelmRelease when feature work lands.

## Ordered implementation steps

### 1. Refresh targeted context and create migration baseline

Owner: `ansible-implementor`

- Re-read ADR-010, ADR-011, ADR-012, ADR-013, and ADR-014.
- Re-read the current files listed in this plan.
- Do not re-audit the verified build status; use it as given.
- Capture a temporary baseline of semantic outputs for comparison:
  - parsed YAML from current group_vars.
  - rendered current Talos patch templates using representative current vars, without touching live Talos.
  - current `kustomize build flux/infrastructure/controllers` output, which should be empty/no-resource.
- Validation: baseline files are local temp artifacts only and not committed.

### 2. Add config input layout and schema validation

Owner: `ansible-implementor`

- Add `config/defaults/cluster.yaml`, `config/overrides/cluster.yaml`, and `config/overrides/nodes.yaml`.
- Migrate all current non-secret values from common/hcloud group_vars into overrides without loss.
- Add validation tasks for merge rules, computed-key rejection, provider-gated features, and current feature constraints.
- Keep `hcloud.sops.yaml` untouched.
- Validation:
  - `ansible-playbook playbooks/render-config.yml --syntax-check` from `infra/ansible/` once the playbook exists.
  - Render validation fails clearly on invalid schema, not with template stack traces.

### 3. Implement render playbook and role

Owner: `ansible-implementor`

- Add `infra/ansible/playbooks/render-config.yml` and `infra/ansible/roles/config_render/`.
- Implement deterministic deep merge and computed values using built-in Ansible/Jinja filters where possible; avoid new dependencies unless justified.
- Emit generated group_vars with current variable names.
- Add check/drift mode that renders to a temp directory and diffs against committed generated files.
- Validation:
  - `ansible-lint`.
  - `ansible-playbook playbooks/render-config.yml --syntax-check`.
  - Render twice and confirm no diff on the second run.

### 4. Refactor Talos patch generation to layered model

Owner: `ansible-implementor`

- Generate ordered patch templates/fragments from `cluster.yaml` and `nodes.yaml`.
- Update `talos_config` to discover/use ordered all-node, control-plane, and per-node patch lists instead of fixed `common.yaml.j2` and `controlplane.yaml.j2` names.
- Preserve current inline hostname/node-label/hybrid scheduling behavior.
- Support `talos.patches.all`, `talos.patches.controlplane`, and `nodes[].patch` as raw fragments.
- Support network `deviceSelector` schema, but preserve `interface: eth1` for kube1 until a safe selector is supplied/confirmed.
- Validation:
  - `ansible-lint`.
  - `ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check`.
  - `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check`.
  - `talosctl validate --config <generated-controlplane-config>` / equivalent local validation if generated machine configs can be safely produced from existing local secrets; otherwise document why validation is limited to patch render/syntax.
  - No live `talosctl apply-config`, `talosctl bootstrap`, or `talosctl upgrade`.

### 5. Add provider bootstrap variable output

Owner: `ansible-implementor`

- Render hcloud provider group vars with `bootstrap_pre_flux_components: [cilium, hcloud-ccm]`.
- Add/render manual provider group vars with `bootstrap_pre_flux_components: [cilium]`.
- Ensure current hcloud variables remain semantically identical.
- Do not implement `flux_bootstrap` feature installation tasks.
- Validation:
  - Provider inventory syntax still loads for manual and hcloud where possible without contacting live APIs beyond normal dynamic inventory constraints.
  - `ansible-playbook ... playbooks/bootstrap-flux.yml --syntax-check` for manual and hcloud.

### 6. Add Flux catalog/base/patches layout

Owner: `k8s-implementor`

- Create `flux/infrastructure/controllers/_components/`, `base/`, and `patches/` layout.
- Make `base/kustomization.yaml` render-owned and currently sparse/empty.
- Keep `controllers/kustomization.yaml` as the user-owned overlay referencing `base/` and prepared for `patches/`.
- Add README guidance explaining that `_components/` catalog entries introducing resources are ordinary kustomize bases, not official `kind: Component` entries.
- Preserve `flux/clusters/kube1/infrastructure.yaml` path and dependency ordering.
- Validation:
  - `kustomize build flux/infrastructure/controllers`.
  - `kustomize build flux/infrastructure/configs`.
  - `kustomize build flux/clusters/kube1` if available/meaningful for Flux CRDs in the local toolchain.
  - No `flux reconcile`, no cluster access.

### 7. Behavior-preserving equivalence gate

Owner: `ansible-implementor` + `k8s-implementor`

- Compare generated group_vars semantics against the old hand-authored values.
- Compare generated Talos patch semantics against old rendered patch semantics.
- Compare Flux rendered output against the prior empty-placeholder behavior.
- Differences allowed without user decision: generated headers, comment movement, path/layout scaffolding that renders to no additional Kubernetes resources.
- Differences requiring explicit user decision: changed Talos machine config values, new cloud-provider flag, changed CNI/proxy settings, changed hcloud provisioning values, enabled Flux resources, or any new live-cluster behavior.
- Validation: produce a short equivalence note in the implementation handoff or reviewer notes.

### 8. Documentation update

Owner: `docs-writer`

- Add/update a runbook explaining the new workflow:
  1. edit `config/overrides/cluster.yaml` or `config/overrides/nodes.yaml`;
  2. run the render playbook;
  3. review generated diffs;
  4. run validation;
  5. only then run live bootstrap/upgrade playbooks if the generated diff requires it.
- Explain ownership boundaries: template/defaults vs user/overrides vs render/generated.
- Explain that Cilium/CCM/CSI/Flux-bootstrap remain deferred.
- Update `CLAUDE.md` implementation status only if the user wants the repository status section updated as part of the same implementation slice; otherwise list it as a follow-up.
- Validation: docs review for consistency with file names and no claims that deferred features are built.

### 9. ADR follow-up decision point

Owner: live orchestrator + user; `adr-writer` only after agreement

- Present the ADR-005 vs ADR-010 conflict and the proposal that ADR-010 supersedes ADR-005 for CCM steady-state ownership.
- If user confirms Flux adoption: assign `adr-writer` to update/cross-reference ADR-005 or add a supersession note in the appropriate ADR location.
- If user confirms CCM remains outside Flux: assign `adr-writer` to update ADR-010/012 expectations and ensure `_components/providers/hcloud/` does not model CCM.
- Do not let implementors decide this in code.

### 10. Review lanes

Owners: `ansible-reviewer` and `k8s-reviewer`

Ansible review scope:

- One-writer-per-file invariant and generated headers.
- Idempotent render behavior.
- Schema validation quality.
- No secret rendering/decryption/logging.
- Talos patch ordering and no generated machine configs committed.
- `bootstrap_pre_flux_components` provider wiring.
- Required validation: `ansible-lint`, render syntax check, bootstrap syntax checks, and safe Talos validation or documented limitation.

Kubernetes/Flux review scope:

- `_components`/`base`/`patches` ownership and Kustomize correctness.
- No accidental resources for deferred features.
- `flux/clusters/kube1/infrastructure.yaml` ordering preserved.
- Required validation: `kustomize build` for touched Flux paths.

Route findings back to the matching implementor. Reviewers do not edit.

### 11. Closing report

Owner: `report-writer`

- After implementation and review pass, write `.ai/reports/2026-06-22-refactor-render-architecture.md`.
- Include what changed, equivalence results, validation run, deferred feature work, live-cluster commands the user still must run if any, and ADR follow-ups.

## Validation required before handoff

No live cluster operations are required or allowed for this refactor without explicit user approval.

From `infra/ansible/`:

```bash
ansible-lint
ansible-playbook playbooks/render-config.yml --syntax-check
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-flux.yml --syntax-check
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check
```

From repo root:

```bash
kustomize build flux/infrastructure/controllers
kustomize build flux/infrastructure/configs
kustomize build flux/clusters/kube1
```

Talos validation:

```bash
talosctl validate --config <generated-controlplane-config>
talosctl validate --config <generated-worker-config>
```

Only run Talos validation against locally generated config files if the required local secrets already exist and the command does not contact the live cluster. Never commit the generated configs. If local secrets are unavailable, document this and validate the generated patches by render/equivalence checks instead.

Equivalence checks:

- Render to a temp directory and compare generated group_vars semantics to the pre-refactor YAML values.
- Render Talos patch templates with representative current vars and compare semantic YAML to current `common.yaml.j2` and `controlplane.yaml.j2` output.
- Confirm `kustomize build flux/infrastructure/controllers` still produces no controller resources until feature work lands.
- Run the renderer twice and confirm no second-run diff.

Commands not to run during this plan:

- `ansible-playbook ... provision-infra.yml`
- `ansible-playbook ... image-talos.yml`
- `ansible-playbook ... bootstrap-talos.yml` without `--syntax-check`
- `ansible-playbook ... bootstrap-flux.yml` without `--syntax-check`
- `talosctl apply-config`, `talosctl bootstrap`, `talosctl upgrade`
- `kubectl`, `flux reconcile`, `hcloud` live operations

## Risks, side effects, cost, and live-cluster boundaries

- **Generated drift risk:** If users edit generated files after this refactor, render will overwrite them. Mitigate with generated headers, docs, and render check mode.
- **Semantic drift risk:** Moving from hand-authored patches/group_vars to generated files can accidentally change Talos or provisioning behavior. Mitigate with behavior-preserving equivalence as a hard gate.
- **Talos patch order risk:** Later patches win. Incorrect ordering can erase CNI/proxy/VIP/SAN settings. Keep patch order explicit and reviewer-audited.
- **Kustomize component terminology risk:** Official Kustomize `kind: Component` cannot introduce resources. Treat `_components/` as a catalog of ordinary bases for controllers unless an entry is transform-only.
- **Device selector risk:** ADR-013 prefers `deviceSelector`, but guessing a selector for current Hetzner nodes could break networking. Preserve `eth1` until a safe selector is confirmed or per-node hardware data is supplied.
- **Secret boundary:** Do not render or decrypt `hcloud.sops.yaml`; do not write hcloud tokens into generated files.
- **Dynamic inventory boundary:** ADR-014 hcloud inventory contacts Hetzner when loaded. Syntax checks that load hcloud inventory may need the SOPS token path but must not create/change resources.
- **No cost-incurring operations:** This refactor should not create servers, snapshots, volumes, load balancers, or PVCs.
- **No live-cluster changes:** Agents must not run live provisioning/bootstrap/Talos/Kubernetes/Flux commands without explicit user approval.

## ADR needs

- **ADR update likely required after confirmation:** Resolve the ADR-005 vs ADR-010 CCM steady-state ownership conflict. Recommended outcome is ADR-010 supersedes ADR-005 for Flux adoption, but this needs live-session/user confirmation.
- **No new ADR needed for the compiler itself:** ADR-011 through ADR-013 already record the architecture being implemented.
- **No ADR change for ADR-014:** It is built and should be referenced, not rewritten.
- **Possible ADR note if Kustomize `Component` is rejected for controller catalog entries:** If the team expected literal `kind: Component`, record a short ADR amendment or note explaining that controller catalog entries are ordinary kustomize bases because official Components cannot contain resources.

## Open questions for live-session resolution

All questions resolved (user-confirmed 2026-06-22):

1. **CCM steady-state ownership — RESOLVED:** ADR-010 supersedes ADR-005. CCM is pre-Flux installed, Flux-adopted. `adr-writer` to update ADR-005 with supersession note.
2. **Talos network interface — RESOLVED:** Probe real hardware via `talosctl` to discover actual interface identifiers (MAC address / PCI vendor-device), build a real `deviceSelector`, and use it in the rendered config. User authorizes live read-only `talosctl` probes and accepts re-bootstrap if required. See "Talos deviceSelector probe" section below.
3. **CLAUDE.md implementation status — RESOLVED:** Maintain CLAUDE.md and other repo artifacts as part of the work via `docs-writer`; no need to ask each time. Update honestly when something becomes built.
4. **Render check mode / CI — RESOLVED:** No CI exists for this repo yet. Provide render check as a local validation command only; skip CI wiring. Can be revisited when CI is introduced.

## Talos deviceSelector probe

Owner: user runs the probe (live read-only) after cluster bootstrap; `ansible-implementor` consumes the output

**Current status (2026-06-22):** The cluster is not currently bootstrapped — `infra/talos/secrets/` is empty, no talosconfig exists. The `talosctl` probe cannot run until the cluster is bootstrapped (provision-infra.yml + bootstrap-talos.yml, which is live cluster work outside this refactor's scope).

**Interim approach:** Step 4 builds full `deviceSelector` schema and rendering support into the config compiler and Talos patch renderer, but uses `interface: eth1` as the current override value (behavior-preserving). When the cluster is bootstrapped (part of later feature work or a separate bootstrap session), the user runs:

```bash
# Network interfaces with MAC addresses and PCI info
talosctl -n <node-ip> get links
talosctl -n <node-ip> get hardwarecomponents
```

The output is then fed to `ansible-implementor`, who derives the `deviceSelector` (likely by MAC address or PCI vendor/device ID for the private-network interface) and wires it into `config/overrides/cluster.yaml` or `config/overrides/nodes.yaml` `talos.network.defaultInterface.deviceSelector`, replacing `interface: eth1`.

If the probe reveals that `eth1` is the only reliable selector (e.g. Hetzner virtualizes interfaces without stable PCI identities), document that and keep `interface: eth1` with a note explaining why `deviceSelector` was not viable. The decision is evidence-driven, not assumed.

**This is a deferred follow-up, not a blocker for the refactor.** The refactor's quality gate is behavior-preserving equivalence with the current `interface: eth1` config.

## Defaults vs overrides principle

**`config/defaults/cluster.yaml` contains only sensible provider-agnostic defaults** — values a new cluster using this template would reasonably start with (e.g. `features.cni: cilium`, a stable `kubernetesVersion`, `talos.version`, empty certSANs, `firewallOpenAccess: false`). **`config/overrides/cluster.yaml` carries all kube1-specific choices** — `name: kube1`, `vip: 10.0.0.10`, `kubernetesVersion: v1.36.0`, `talos.version: v1.13.3`, `provider.name: hcloud`, hcloud topology, server types, network names, etc.

The implementor must not leak kube1-specific values into defaults. If a value is kube1-specific, it belongs in overrides. If a value is a sensible generic default, it belongs in defaults. The migration step copies current hand-authored values into overrides, not defaults.

## Deferred feature work

After this refactor lands and the ADR-005/010 question is resolved, revise `.ai/plans/2026-06-22-cilium-ccm-csi.md` before implementing feature work. That revised plan should consume the new compiler outputs and catalog layout rather than adding direct hand-authored resources to the old placeholder structure.
