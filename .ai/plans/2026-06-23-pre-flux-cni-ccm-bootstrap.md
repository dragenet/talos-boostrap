# 2026-06-23 — Pre-Flux CNI and CCM bootstrap

## Goal and scope

Implement the kube1 pre-Flux bootstrap layer for the two controllers that must exist before Flux can safely reconcile the rest of the cluster:

1. **Cilium CNI** as core bootstrap networking, installed by Ansible before `flux bootstrap`, then adopted by Flux.
2. **Hetzner Cloud Controller Manager (hcloud CCM)** as provider cloud integration, installed by Ansible before `flux bootstrap`, then adopted by Flux.
3. **Config/render migration** from the current transitional `features.cni` / `features.hcloudCcm` shape to the agreed core keys: `cni.name` and `cloud_controller_manager.enabled`.
4. **Docs/status updates** needed to keep CLAUDE.md, runbooks, and ADR implementation notes honest.

Out of scope for this slice:

- Hetzner CSI / StorageClasses / PVC tests.
- cert-manager, OpenEBS, ingress, external-dns, or app workloads.
- Live cluster execution by agents. The user runs live `ansible-playbook`, `kubectl`, `helm`, `flux`, `hcloud`, and `talosctl` commands unless they explicitly approve an agent to do so.

## Current-state facts from repo context

- The config compiler exists under `infra/ansible/playbooks/render-config.yml` and `infra/ansible/roles/config_render/`.
- Current schema is transitional:
  - `config/defaults/cluster.yaml` has `features.cni: cilium` and `features.hcloudCcm: false`.
  - `config/overrides/cluster.yaml` has `features.cni: cilium` and does not set `features.hcloudCcm`.
  - ADR-002 / ADR-005 / ADR-010 already say CNI and CCM are **core bootstrap config**, not optional `features` entries.
- `config_render/tasks/compute.yml` currently computes `bootstrap_pre_flux_components` from `features.cni` and `features.hcloudCcm`.
- Generated hcloud vars currently contain only:
  ```yaml
  bootstrap_pre_flux_components:
    - cilium
  ```
  so hcloud CCM is not currently selected.
- `config_render_cluster_external_cloud_provider_enabled` already gates the generated Talos all-node patch `infra/talos/patches/generated/all/10-cloud-provider.yaml.j2`, but that patch is currently absent because hcloud CCM is gated off.
- Generated Talos control-plane patch `infra/talos/patches/generated/controlplane/10-cni-proxy.yaml.j2` already disables Talos-managed CNI and kube-proxy for Cilium (`cluster.network.cni.name: none`, `cluster.proxy.disabled: true`).
- `infra/ansible/playbooks/bootstrap-flux.yml` only applies the `flux_bootstrap` role.
- `infra/ansible/roles/flux_bootstrap/tasks/main.yml` is a comment-only stub.
- `infra/ansible/requirements.yml` pins `hetzner.hcloud` and `community.sops`; it does not pin `kubernetes.core` yet.
- `flux/infrastructure/controllers/_components/` exists as a sparse catalog; it has no Cilium or hcloud CCM bases yet.
- `flux/infrastructure/controllers/base/kustomization.yaml` is committed as render-owned with `resources: []`, but `config_render/tasks/render_flux.yml` is still a stub. This must be reconciled so future renders populate the base deterministically.
- Flux entrypoint ordering exists: `infrastructure-controllers` waits, `infrastructure-configs` depends on controllers and waits, and `apps` depends on configs.
- `flux_repo_url` is still empty in generated `infra/ansible/inventories/common/group_vars/all/flux.yml`; live Flux bootstrap cannot succeed until the user fills `config/overrides/cluster.yaml` `flux.repoUrl` and re-renders.
- The hcloud API token remains encrypted in `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml`; do not decrypt, print, or copy it into Git manifests.

## Official docs to consult before implementation

Implementors must refresh these official docs immediately before editing or pinning versions:

- Talos / SideroLabs:
  - Talos Cilium guide: https://docs.siderolabs.com/kubernetes-guides/cni/deploying-cilium
  - Talos machine configuration reference for external cloud provider / KubePrism: https://docs.siderolabs.com/talos/v1.13/reference/configuration/v1alpha1/config.md
  - Talos Hetzner platform notes: https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/cloud-platforms/hetzner.md
- Cilium:
  - Helm install docs: https://docs.cilium.io/en/stable/installation/k8s-install-helm/
  - Kube-proxy replacement: https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
  - Helm values reference: https://docs.cilium.io/en/stable/helm-reference/
- Hetzner CCM:
  - Chart README: https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/chart/README.md
  - Quickstart: https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/docs/guides/quickstart.md
  - Helm values reference / chart templates: https://github.com/hetznercloud/hcloud-cloud-controller-manager/tree/main/chart
- Flux:
  - Bootstrap docs: https://fluxcd.io/flux/installation/bootstrap/
  - Generic Git bootstrap: https://fluxcd.io/flux/installation/bootstrap/generic-git-server/
  - HelmRelease spec/adoption-sensitive fields: https://fluxcd.io/flux/components/helm/helmreleases/
  - Kustomization `dependsOn`, `wait`, `prune`, `timeout`: https://fluxcd.io/flux/components/kustomize/kustomizations/
- Ansible:
  - `kubernetes.core.helm`: https://docs.ansible.com/ansible/latest/collections/kubernetes/core/helm_module.html
  - `kubernetes.core.k8s`: https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html

Docs context already revalidated for this plan:

- Cilium supports non-`kube-system` namespaces in current chart/docs; use `cilium-system`.
- hcloud CCM chart templates use `.Release.Namespace`; use `hcloud-ccm-system` and create the `hcloud` Secret in that namespace.
- Pin hcloud CCM to a current release; avoid `<= v1.30.0` because of the Hetzner API `server.datacenter` removal risk after July 2026.
- For Talos + Cilium, verify Talos-specific values before coding, especially kube-proxy replacement, cgroup settings, and whether kube API access should use KubePrism (`localhost:7445`) rather than the VIP.

## Settled decisions

- **Core config, not optional features:** `features` stays for optional platform/app features only. CNI moves to `cni.name`; CCM moves to `cloud_controller_manager.enabled`.
- **CNI selection:** kube1 uses Cilium. Cilium is installed pre-Flux by Ansible, then Flux adopts the same Helm release.
- **CCM selection:** hcloud provider uses hcloud CCM when `cloud_controller_manager.enabled` resolves true/auto. Manual provider does not enable external cloud provider or install a CCM.
- **Namespaces:** pre-Flux components use dedicated `<component>-system` namespaces: `cilium-system` and `hcloud-ccm-system`.
- **Flux handoff:** Ansible installs only what is needed before Flux can schedule. Flux then reconciles/adopts the same release names, namespaces, chart versions, and values.
- **Cilium config intent:** VXLAN, `kubeProxyReplacement: true`, `l7Proxy: false`, `gatewayAPI.enabled: false`.
- **hcloud CCM config intent:** `networking.enabled: false` for kube1 because Cilium uses VXLAN and kube1 does not need CCM-managed pod routes or LoadBalancer services yet.
- **Secret ownership:** Ansible creates/updates the Kubernetes Secret from the encrypted inventory token. Flux references it; Flux does not own a plaintext Secret manifest.

## Still open / must be confirmed before live bootstrap

- `flux.repoUrl` and Flux authentication method are not configured. The user must provide these before live `bootstrap-flux.yml` can run.
- Exact Cilium and hcloud CCM chart versions must be checked against official chart indexes immediately before implementation.
- The implementor must verify whether the non-Cilium schema value is `flannel` per ADR-002 or whether a legacy `none` compatibility path is needed for one release. Do not silently leave `features.cni` as the durable API.
- The implementor must verify Cilium and hcloud CCM tolerations under external cloud provider mode. Cilium and CCM must be schedulable even while nodes carry `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule`.
- If implementation discovers that either Cilium or hcloud CCM cannot safely run in its dedicated namespace, stop and request an architecture decision update before using `kube-system`.

## Ordered implementation plan and task boundaries

### 1. Refresh focused repo/docs context

Owner: live orchestrator with graphify + `web-fast-context`

- Re-run graphify for the exact files in this plan before dispatching implementors.
- Re-check the official docs above for chart versions, required values, and namespace support.
- Confirm no newer ADR/report has superseded ADR-002, ADR-005, ADR-010, ADR-011, ADR-012, or this plan.

Validation before next step: context summary identifies exact files to edit and any doc deltas.

### 2. Migrate core config schema and render computation

Owner: `ansible-implementor`

Bounded file area:

- `config/defaults/cluster.yaml`
- `config/overrides/cluster.yaml`
- `infra/ansible/roles/config_render/tasks/{validate,compute,render_ansible,render_talos}.yml`
- `infra/ansible/roles/config_render/templates/group_vars/**`
- `infra/ansible/roles/config_render/templates/talos_patches/**`
- generated outputs under `infra/ansible/inventories/**/group_vars/` and `infra/talos/patches/generated/`

Work:

- Replace transitional `features.cni` input with `cni.name`.
- Replace transitional `features.hcloudCcm` input with `cloud_controller_manager.enabled` (`auto` / `true` / `false`) per ADR-005.
- Keep `features` limited to optional platform features (`certManager`, `ingress`, `openebs`, `hcloudCsi`, future optional features).
- Compute `bootstrap_pre_flux_components` from `cni.name`, `cloud_controller_manager.enabled`, and `provider.name`.
- Compute `cluster_external_cloud_provider_enabled` only when the selected provider CCM will be installed pre-Flux.
- Add validation that rejects stale top-level `features.cni` / `features.hcloudCcm` with clear migration messages, unless the user explicitly requests a compatibility alias.
- Preserve the one-writer rule: generated group vars and Talos patch files remain render-owned.

Validation before handoff:

- `ansible-playbook playbooks/render-config.yml --syntax-check` from `infra/ansible/`.
- `ansible-playbook playbooks/render-config.yml -e config_render_check=true` from `infra/ansible/`.
- Inspect generated diff: hcloud provider should render `bootstrap_pre_flux_components: [cilium, hcloud-ccm]` only when CCM resolves enabled; manual must not render hcloud CCM.
- Confirm the all-node external cloud provider Talos patch appears only when hcloud CCM is selected.

### 3. Add Ansible Kubernetes dependencies and role defaults

Owner: `ansible-implementor`

Bounded file area:

- `infra/ansible/requirements.yml`
- `infra/ansible/roles/flux_bootstrap/defaults/main.yml` (new if absent)
- optional role var docs/comments under `infra/ansible/roles/flux_bootstrap/`

Work:

- Pin `kubernetes.core` if the role uses `kubernetes.core.k8s` / `kubernetes.core.helm`.
- Add role-local `flux_bootstrap_*` defaults for component namespaces, release names, chart repos, chart versions, timeouts, and values.
- Keep cluster selection variables sourced from rendered group vars (`bootstrap_pre_flux_components`, `cluster_endpoint`, hcloud vars), not duplicated.

Validation before handoff:

- `ansible-galaxy collection install -r requirements.yml` only if dependency installation is needed locally.
- `ansible-lint` from `infra/ansible/`.
- No unpinned collection versions.

### 4. Implement preflight and namespace tasks for `flux_bootstrap`

Owner: `ansible-implementor`

Bounded file area:

- `infra/ansible/roles/flux_bootstrap/tasks/main.yml`
- `infra/ansible/roles/flux_bootstrap/tasks/preflight.yml`
- `infra/ansible/roles/flux_bootstrap/tasks/namespaces.yml`

Work:

- Split the stub role into included task files.
- Assert required local tools/config exist (`kubectl`, `helm`, `flux`, usable kubeconfig path/context) without contacting live systems beyond what is explicitly allowed.
- Assert `flux_repo_url` is non-empty before the Flux bootstrap task runs.
- Assert `hcloud_token` is present when `hcloud-ccm` is selected, with `no_log: true` around secret-sensitive checks.
- Create/ensure `cilium-system` and `hcloud-ccm-system` namespaces as needed during live execution.

Validation before handoff:

- `ansible-lint`.
- `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check`.
- Do not run the playbook live or in check mode without user approval because it targets the Kubernetes API.

### 5. Implement Cilium pre-Flux install

Owner: `ansible-implementor`

Bounded file area:

- `infra/ansible/roles/flux_bootstrap/tasks/cilium.yml`
- `infra/ansible/roles/flux_bootstrap/defaults/main.yml`

Work:

- Install/upgrade Helm release `cilium` in namespace `cilium-system` when `cilium` is in `bootstrap_pre_flux_components`.
- Use the same release name, namespace, chart version, and values planned for the Flux HelmRelease.
- Apply Talos/Cilium-required values after rechecking docs: kube-proxy replacement, VXLAN, L7/Gateway disabled, cgroup handling, capabilities, and kube API service host/port.
- Wait for readiness with a conservative timeout.
- Add clear comments explaining why Cilium is pre-Flux: Flux controllers are pods and need pod networking first.

Validation before handoff:

- Static Ansible lint/syntax checks.
- Render the equivalent Helm template locally if feasible (`helm template`) using the pinned values.
- Reviewer must verify Cilium chart supports `cilium-system` for the pinned version.
- Live validation commands are documented for the user, not run by the agent: Cilium pods ready and kube-proxy replacement status true.

### 6. Implement hcloud CCM pre-Flux install

Owner: `ansible-implementor`

Bounded file area:

- `infra/ansible/roles/flux_bootstrap/tasks/hcloud_ccm.yml`
- `infra/ansible/roles/flux_bootstrap/defaults/main.yml`

Work:

- Create/update Secret `hcloud-ccm-system/hcloud` from `hcloud_token` with `no_log: true`.
- Install/upgrade Helm release `hcloud-cloud-controller-manager` in namespace `hcloud-ccm-system` when `hcloud-ccm` is in `bootstrap_pre_flux_components`.
- Use `networking.enabled: false` unless official docs or an agreed architecture update require otherwise.
- Wait for deployment readiness.
- Add a post-install readiness check that nodes no longer have the cloud-provider uninitialized taint and have `spec.providerID` values beginning with `hcloud://`.

Validation before handoff:

- Static Ansible lint/syntax checks.
- Local Helm template check if feasible.
- Verify no task logs `hcloud_token`, places it in command args, or writes it to an unencrypted file.
- Live node-taint/providerID checks are documented for the user, not run by the agent.

### 7. Implement Flux bootstrap command boundary

Owner: `ansible-implementor`

Bounded file area:

- `infra/ansible/roles/flux_bootstrap/tasks/flux.yml`
- `infra/ansible/roles/flux_bootstrap/tasks/main.yml`
- `infra/ansible/playbooks/bootstrap-flux.yml` comments if needed

Work:

- Run `flux bootstrap git` only after selected pre-Flux components are installed and ready.
- Use rendered `flux_repo_url`, `flux_repo_branch`, and `flux_path`.
- Make side effects explicit in task names/comments: this command contacts the cluster and may commit/push Flux manifests to the configured Git remote.
- Keep authentication method configurable; do not hardcode secrets.

Validation before handoff:

- Syntax/lint only unless user explicitly approves live execution.
- Document the exact user command and prerequisites for live bootstrap.

### 8. Add Flux catalog base for Cilium adoption

Owner: `k8s-implementor`

Bounded file area:

- `flux/infrastructure/controllers/_components/cilium/`

Work:

- Add a self-contained ordinary kustomize base for Cilium.
- Include Namespace `cilium-system` if appropriate for steady-state ownership.
- Add HelmRepository/HelmRelease resources with exact release name, target namespace, chart version, and values matching the Ansible pre-Flux install.
- Ensure adoption-sensitive fields match Ansible exactly; do not change release name or namespace.

Validation before handoff:

- `kustomize build flux/infrastructure/controllers/_components/cilium`.
- `flux build` or equivalent render validation if available.
- Optional local `helm template` for the pinned chart/values.

### 9. Add Flux catalog base for hcloud CCM adoption

Owner: `k8s-implementor`

Bounded file area:

- `flux/infrastructure/controllers/_components/providers/hcloud/ccm/`

Work:

- Add a self-contained ordinary kustomize base for hcloud CCM.
- Include Namespace `hcloud-ccm-system` if appropriate for steady-state ownership.
- Add HelmRepository/HelmRelease resources with exact release name, target namespace, chart version, and values matching the Ansible pre-Flux install.
- Reference the pre-created Secret by name/key; do not add a plaintext Secret manifest.
- Add HelmRelease dependency on Cilium if Flux dependency semantics support it cleanly for these resources; otherwise rely on the already-running pre-Flux install and document the rationale.

Validation before handoff:

- `kustomize build flux/infrastructure/controllers/_components/providers/hcloud/ccm`.
- `flux build` or equivalent render validation if available.
- Verify no hcloud token appears in Git.

### 10. Implement Flux base rendering for selected core components

Owner: `ansible-implementor`

Bounded file area:

- `infra/ansible/roles/config_render/tasks/render_flux.yml`
- `infra/ansible/roles/config_render/templates/flux/**` (new if needed)
- generated `flux/infrastructure/controllers/base/kustomization.yaml`
- generated `flux/infrastructure/controllers/base/cluster-vars.yaml` if populated

Work:

- Replace the current `render_flux.yml` stub with deterministic rendering.
- Render `base/kustomization.yaml` resources from core config and provider selection:
  - include `_components/cilium` when `cni.name: cilium`;
  - include `_components/providers/hcloud/ccm` when hcloud CCM resolves enabled;
  - leave optional `features` for later platform components.
- Preserve generated headers.
- Keep the user-owned overlay `flux/infrastructure/controllers/kustomization.yaml` unchanged except if docs/comments need repair.

Validation before handoff:

- `ansible-playbook playbooks/render-config.yml -e config_render_check=true`.
- `kustomize build flux/infrastructure/controllers`.
- Confirm rendered base contains only selected components and no stale resources.

### 11. Review Ansible lane

Owner: `ansible-reviewer`

Scope:

- Config schema/render changes.
- `flux_bootstrap` role.
- Ansible requirements.
- Generated Ansible/Talos outputs.

Validation expectations:

- Idempotency, FQCN usage, variable namespacing, no-log secret handling.
- No live-cluster/provider-costing commands unless approved.
- `ansible-lint` pass.
- Render syntax/check pass.
- `bootstrap-flux.yml --syntax-check` pass for hcloud and manual inventories where applicable.

### 12. Review Kubernetes/Flux lane

Owner: `k8s-reviewer`

Scope:

- Cilium and hcloud CCM catalog bases.
- Rendered `flux/infrastructure/controllers/base` output.
- Cluster entrypoint dependency/wait/prune behavior.

Validation expectations:

- Chart versions pinned.
- Release names/namespaces/values match Ansible pre-Flux installs.
- Kustomize render passes.
- Flux adoption hazards called out.
- No plaintext hcloud token or accidental Secret ownership by Flux.

### 13. Docs and status updates

Owner: `docs-writer`

Bounded file area:

- `CLAUDE.md`
- existing bootstrap/runbook docs under `docs/`
- possibly `README.md` if it has bootstrap steps

Work:

- Update implementation status: Cilium/hcloud CCM pre-Flux install and Flux adoption are implemented only after review passes.
- Update bootstrap workflow to explain:
  - edit config → render → bootstrap Talos → bootstrap Flux;
  - Cilium before Flux;
  - external cloud provider taint and hcloud CCM clearing it;
  - dedicated namespaces;
  - live-cluster side effects.
- Add user-facing validation commands for post-bootstrap health.

Validation before handoff:

- Docs accurately distinguish implemented vs planned components.
- No instructions tell agents to run live commands without user approval.

### 14. ADR/status cleanup

Owner: `adr-writer` after user confirms no architecture changes were introduced beyond the settled decisions

Likely file area:

- `docs/cluster-bootstrap/adrs/002-cilium-cni.md`
- `docs/cluster-bootstrap/adrs/005-cloud-controller-manager-pattern.md`
- `docs/cluster-bootstrap/adrs/010-ansible-flux-provider-handoff.md`
- `docs/cluster-bootstrap/adrs/012-feature-catalog.md`

Work:

- Update implementation-status notes after code lands.
- Fix any stale wording that still treats CNI or CCM as optional `features` entries.
- Do not create a new ADR unless implementation changes a durable decision, such as using `kube-system`, changing the pre-Flux/Flux-adoption boundary, or enabling hcloud CCM networking/routes.

Validation before handoff:

- ADRs agree with each other on core config keys, namespaces, and Ansible-to-Flux handoff.

### 15. Closing report

Owner: `report-writer`

- After implementation, review, and docs/ADR updates, write `.ai/reports/2026-06-23-pre-flux-cni-ccm-bootstrap.md`.
- Include changed files, validation results, live commands the user still needs to run, and open follow-ups.

## Validation matrix

Local/static validation expected before user handoff:

- `ansible-lint` from `infra/ansible/`.
- `ansible-playbook playbooks/render-config.yml --syntax-check` from `infra/ansible/`.
- `ansible-playbook playbooks/render-config.yml -e config_render_check=true` from `infra/ansible/`.
- `ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check` from `infra/ansible/`.
- Manual-provider syntax check if schema changes affect manual inventory.
- `kustomize build flux/infrastructure/controllers`.
- `kustomize build` for each new `_components/...` base.
- `flux build` / Flux manifest validation if available locally.
- Local `helm template` for Cilium and hcloud CCM with the exact pinned values if chart repos are reachable.

Live validation for the user to run after approval / implementation:

- Re-render config and inspect generated diffs.
- Re-apply Talos config if the external cloud provider patch is newly emitted.
- Run `bootstrap-flux.yml` against the selected provider inventory.
- Check Cilium pods in `cilium-system` are Ready and kube-proxy replacement is enabled.
- Check hcloud CCM in `hcloud-ccm-system` is Ready.
- Check nodes have no `node.cloudprovider.kubernetes.io/uninitialized` taint and have `spec.providerID: hcloud://...`.
- Check Flux controllers in `flux-system` are Ready and `infrastructure-controllers` reconciles.

## Risks, side effects, and live-cluster boundaries

- **External cloud provider taint:** Enabling Talos `externalCloudProvider` before hcloud CCM is ready can leave nodes tainted `NoSchedule`. Keep config migration, CCM installer, and Flux adoption in one reviewed implementation slice; do not flip the gate early on a live cluster.
- **CNI race:** Flux controllers cannot run without a CNI. Cilium must be installed and ready before `flux bootstrap` installs controllers.
- **Namespace adoption sensitivity:** Ansible and Flux must use the same release names, namespaces, chart versions, and values. A mismatch can cause Flux/Helm to create a second release or change the existing one unexpectedly.
- **Secret handling:** The hcloud token is provider API access. Use `no_log: true`; never put it in command args, diffs, logs, unencrypted vars, or Flux manifests.
- **Hetzner API side effects:** hcloud CCM talks to Hetzner Cloud and may mutate node metadata / provider integration state. With `networking.enabled: false` and no LoadBalancer services, this slice should not create paid load balancers, routes, volumes, or servers.
- **Flux bootstrap side effects:** `flux bootstrap git` modifies the live cluster and can write/commit/push manifests to the configured Git remote.
- **No provider-costing commands by agents:** Agents must not run provisioning, `hcloud`, live `kubectl`, live `helm`, live `flux`, `talosctl`, or bootstrap playbooks without explicit user approval.
- **Chart version drift:** Re-check Cilium and hcloud CCM chart versions before pinning. Avoid hcloud CCM versions affected by the July 2026 Hetzner API deprecation.
- **Manual provider boundary:** Manual/bare-metal must not emit hcloud CCM or external cloud provider patches. It may use Cilium if selected.

## ADR needs

- No new ADR is expected if implementation follows the settled decisions above.
- `adr-writer` should update stale implementation-status notes and any inconsistent taxonomy text after implementation.
- Stop for a user decision and plan/ADR update if implementation needs to change any durable choice: namespace convention, CNI/CCM core key names, pre-Flux vs Flux-only boundary, hcloud CCM networking/routes, or Secret ownership.
