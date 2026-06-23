# 2026-06-22 — Cilium, Hetzner CCM, and Hetzner CSI bootstrap

## Goal and scope

Make kube1 ready for real Kubernetes workloads by adding the required networking and Hetzner cloud-integration layer, then handing steady-state ownership to Flux:

1. **Cilium** as the pre-Flux CNI with kube-proxy replacement.
2. **Hetzner Cloud Controller Manager (CCM)** as a pre-Flux bootstrap component after Talos is configured for an external cloud provider.
3. **Hetzner CSI Driver** as Flux-managed block storage for PVCs backed by Hetzner Cloud Volumes.

This plan spans Talos machine config patches, Ansible bootstrap, Flux HelmReleases/Kustomizations, docs, and ADR follow-up. It must not be implemented as one undifferentiated change: keep Ansible/Talos work and Flux/Kubernetes work in separate subagent lanes, then review both.

## Install-boundary decision

| Component | Install boundary | Steady-state owner | Rationale |
|---|---|---|---|
| Talos external cloud provider flag | Ansible/Talos via `bootstrap-talos.yml` before `bootstrap-flux.yml` | Talos machine config in Git | Required before hcloud CCM can initialize nodes. This is a bootstrap-boundary setting, not a Helm resource. |
| Cilium | **Pre-Flux Ansible Helm install** | Flux adopts the same Helm release | Pods, including Flux controllers, cannot schedule/network without a CNI. Kube-proxy is already disabled in Talos, so Cilium must be present before Flux starts reconciling. |
| Hetzner CCM | **Pre-Flux Ansible Helm install** | Flux adopts the same Helm release | With external cloud provider enabled, Kubernetes taints nodes with `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule`. CCM must run before Flux so it can clear the taint and set provider IDs. |
| Hetzner CSI | **Flux-managed from first reconciliation** | Flux installs and owns it | CSI is not required for Flux controllers to schedule. Installing it after Cilium + CCM through Flux preserves day-one storage completeness while keeping the bootstrap layer minimal and matching existing ADR-010 / kube1 ADR-005 intent. |

Alternative for CSI: install pre-Flux and let Flux adopt it too. This gives storage a few minutes earlier but expands the bootstrap layer and creates one more component that must be upgraded by Ansible if Flux bootstrap fails. Do **not** choose that unless the user explicitly overrides this plan.

## Current-state facts from repo context

- `infra/ansible/playbooks/bootstrap-flux.yml` currently only applies the `flux_bootstrap` role; it is a stub.
- `infra/ansible/roles/flux_bootstrap/tasks/main.yml` is a comment-only stub.
- `infra/talos/patches/controlplane.yaml.j2` already sets `cluster.network.cni.name: none` and `cluster.proxy.disabled: true`; the comment expects Cilium to be installed pre-Flux.
- `infra/talos/patches/common.yaml.j2` currently sets the installer image and optional Talos API SANs; it does not set `cluster.externalCloudProvider`.
- `inventories/common/group_vars/all/flux.yml` has `flux_repo_url: ''`, so Flux bootstrap cannot run until the real Git URL and auth path are provided.
- `inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml` exposes the encrypted variable name `hcloud_token`; this is the token source for Kubernetes Secret creation. Do not decrypt or print it.
- `inventories/providers/hcloud/group_vars/all/hcloud.yml` defines the provider location/network/topology: `nbg1`, private network `kube1`, `10.0.0.0/16`, subnet `10.0.0.0/24`, and 3 hybrid nodes.
- `infra/ansible/requirements.yml` currently pins `hetzner.hcloud` and `community.sops`; it does not yet pin `kubernetes.core`, which is needed if the role uses Ansible modules for Kubernetes Secrets and Helm releases.
- `flux/clusters/kube1/infrastructure.yaml` already has two Flux Kustomizations: `infrastructure-controllers` with `wait: true`, and `infrastructure-configs` depending on controllers with `wait: true`.
- `flux/infrastructure/controllers/kustomization.yaml`, `flux/infrastructure/configs/kustomization.yaml`, and `flux/apps/kustomization.yaml` are empty placeholders.
- Existing ADRs already shape this work:
  - `docs/cluster-bootstrap/adrs/002-cilium-cni.md`: Cilium default is VXLAN, `kubeProxyReplacement: true`, `l7Proxy: false`, `gatewayAPI.enabled: false`.
  - `docs/cluster-bootstrap/adrs/005-cloud-controller-manager-pattern.md`: cloud providers use external CCM pre-Flux; manual/metal does not.
  - `docs/cluster-bootstrap/adrs/010-ansible-flux-provider-handoff.md`: Cilium and hcloud CCM are pre-Flux and adopted by Flux; Hetzner CSI is Flux-managed/provider-gated.
  - `docs/kube1/adrs/005-hetzner-csi.md`: Hetzner CSI depends on hcloud CCM/provider IDs and is an infrastructure controller.
- There is a small ADR inconsistency to resolve: `cluster-bootstrap/adrs/005-cloud-controller-manager-pattern.md` says CCM remains outside Flux, while ADR-010 and this plan use Flux adoption. Flag this in ADR follow-up.

## Official docs consulted / to re-check before implementation

Implementors must re-check these official sources immediately before coding or pinning versions:

- Talos external cloud provider on Hetzner: https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/cloud-platforms/hetzner.md
- Talos machine config reference for `cluster.externalCloudProvider`: https://docs.siderolabs.com/talos/v1.13/reference/configuration/v1alpha1/config.md
- Cilium kube-proxy replacement Helm install: https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
- Cilium Helm repository / chart versions: https://helm.cilium.io
- Hetzner CCM README: https://github.com/hetznercloud/hcloud-cloud-controller-manager
- Hetzner CCM quickstart: https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/docs/guides/quickstart.md
- Hetzner CCM chart values: https://github.com/hetznercloud/hcloud-cloud-controller-manager/blob/main/chart/values.yaml
- Hetzner CSI README: https://github.com/hetznercloud/csi-driver
- Hetzner CSI quickstart: https://github.com/hetznercloud/csi-driver/blob/main/docs/kubernetes/guides/quickstart.md
- Hetzner CSI chart values: https://github.com/hetznercloud/csi-driver/blob/main/chart/values.yaml
- Flux HelmRelease adoption/spec: https://fluxcd.io/flux/components/helm/helmreleases/
- Flux Helm Operator migration/adoption pattern: https://fluxcd.io/flux/migration/helm-operator-migration/
- Flux HelmRelease dependencies: https://fluxcd.io/flux/components/helm/helmreleases/#dependencies
- Flux Kustomization dependencies/wait/prune/interval: https://fluxcd.io/flux/components/kustomize/kustomizations/
- Flux bootstrap git CLI: https://fluxcd.io/flux/cmd/flux_bootstrap_git/
- Flux generic Git bootstrap guide: https://fluxcd.io/flux/installation/bootstrap/generic-git-server/
- Ansible `kubernetes.core.k8s`: https://docs.ansible.com/ansible/latest/collections/kubernetes/core/k8s_module.html
- Ansible `kubernetes.core.helm`: https://docs.ansible.com/ansible/latest/collections/kubernetes/core/helm_module.html
- `kubernetes.core` collection README: https://github.com/ansible-collections/kubernetes.core#readme

Version pins from docs context as of 2026-06-22, subject to final doc/chart-index confirmation before implementation:

- Cilium chart: `1.19.5`
- hcloud CCM chart: `1.33.0` / upstream release `v1.33.0`; use at least `1.30.1` because older versions are affected by Hetzner API deprecation around `server.datacenter` after July 2026.
- hcloud CSI chart: `2.21.2` / upstream release `v2.21.2`
- `kubernetes.core` Ansible collection: pin a stable release such as `6.4.0` after confirming compatibility with local `ansible-core`.

## Machine config patch design

Owner: `ansible-implementor`

Add a provider-driven, conditional external cloud provider block to `infra/talos/patches/common.yaml.j2` rather than hard-coding it for every provider. Manual/bare-metal inventories must not get this setting because there is no CCM to clear the uninitialized taint.

Target patch shape:

```jinja
{% if cluster_external_cloud_provider_enabled | default(false) | bool %}
cluster:
  externalCloudProvider:
    enabled: true
{% endif %}
```

Variable plan:

- Add `cluster_external_cloud_provider_enabled: false` in common group vars or role defaults.
- Override to `true` for the hcloud provider inventory.
- Keep provider-specific CCM chart details under hcloud provider vars or `flux_bootstrap_*` defaults, not in generic Talos role internals.

Talos docs state this field causes Talos to set `--cloud-provider=external` for kubelet and kube-controller-manager; do not add raw component `extraArgs` unless the docs change.

## Pre-Flux bootstrap role design

Owner: `ansible-implementor`

Implement the existing `infra/ansible/roles/flux_bootstrap` role rather than creating a new role. This keeps the existing workflow command (`playbooks/bootstrap-flux.yml`) stable and matches the current repo comments.

Recommended task split inside the role:

1. `preflight.yml`
   - Assert `flux_repo_url` is non-empty if Flux bootstrap will run.
   - Assert required local tools are present: `kubectl`, `helm`, `flux`.
   - Assert hcloud token exists when `hcloud-ccm` is in the pre-Flux component list.
   - Explain/live-boundary comments: this role talks to the Kubernetes API and may install controllers into the cluster.
2. `hcloud-secret.yml`
   - Create/update Kubernetes Secret `kube-system/hcloud` with key `token` from `hcloud_token` using `kubernetes.core.k8s` and `no_log: true`.
   - Do **not** commit a plaintext Flux Secret. Flux HelmReleases for hcloud CCM and CSI reference this bootstrap-created Secret.
   - Do not add the `network` key unless route-controller networking is enabled later; this plan keeps CCM `networking.enabled: false` because Cilium uses VXLAN.
3. `cilium.yml`
   - Install Helm release `cilium` in `kube-system` from `https://helm.cilium.io`, chart `cilium`, pinned chart version.
   - Required values:
     - `kubeProxyReplacement: true`
     - `k8sServiceHost: "{{ cluster_vip }}"` or the host parsed from `cluster_endpoint`
     - `k8sServicePort: 6443`
     - `routingMode: vxlan`
     - `l7Proxy: false`
     - `gatewayAPI.enabled: false`
   - Use `wait: true` and a conservative timeout so the next step does not race the CNI.
4. `hcloud-ccm.yml`
   - Install Helm release `hcloud-cloud-controller-manager` in `kube-system` from `https://charts.hetzner.cloud`, chart `hcloud-cloud-controller-manager`, pinned chart version.
   - Values:
     - reference Secret `hcloud` key `token` according to the chart values
     - `networking.enabled: false`
   - Wait for readiness and then poll nodes until the `node.cloudprovider.kubernetes.io/uninitialized` taint is absent and `spec.providerID` starts with `hcloud://`.
5. `flux.yml`
   - Run `flux bootstrap git --url={{ flux_repo_url }} --branch={{ flux_repo_branch }} --path={{ flux_path }}` with the agreed authentication method.
   - Treat it as idempotent but side-effecting: it commits/pushes Flux manifests to the Git repo and installs Flux controllers in-cluster.

Variable naming:

- Use role-local `flux_bootstrap_*` names for role defaults.
- Use a provider-driven list such as `flux_bootstrap_pre_flux_components`:
  - common/manual default: `['cilium']` when Cilium is selected, no CCM
  - hcloud override: `['cilium', 'hcloud-ccm']`
- Keep hcloud token as `hcloud_token` from SOPS group vars; do not copy it into unencrypted vars.

Dependency change:

- Add pinned `kubernetes.core` to `infra/ansible/requirements.yml` if using `kubernetes.core.k8s` and `kubernetes.core.helm`.
- Document required Python packages for `kubernetes.core.k8s` (`kubernetes`, `PyYAML`, `jsonpatch`) if not already present in prerequisites.

## Flux GitOps layout and adoption design

Owner: `k8s-implementor`

Keep `flux/clusters/kube1/infrastructure.yaml` as the high-level controllers → configs → apps ordering unless implementation discovers a reason to split it. Use HelmRelease-level dependencies for controller ordering inside `flux/infrastructure/controllers`.

Recommended files:

```text
flux/infrastructure/controllers/
  kustomization.yaml
  cilium.yaml
  hcloud-cloud-controller-manager.yaml
  hcloud-csi.yaml
```

Recommended `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - cilium.yaml
  - hcloud-cloud-controller-manager.yaml
  - hcloud-csi.yaml
```

HelmRelease ownership/adoption rules:

- Use the exact same `releaseName`, `targetNamespace`, chart, chart version, and values as the Ansible Helm installs for Cilium and CCM.
- Do not set `install.disableTakeOwnership: true`; the default permits Flux to adopt the existing Helm release.
- Avoid changing `releaseName` or `targetNamespace` later because Flux/Helm may uninstall the old release and install a new one.
- Add `driftDetection.mode: enabled` if the implementor confirms it is appropriate for these charts.

Ordering:

- `cilium` HelmRelease has no HelmRelease dependency.
- `hcloud-cloud-controller-manager` HelmRelease depends on `cilium`.
- `hcloud-csi` HelmRelease depends on `hcloud-cloud-controller-manager`.
- `infrastructure-configs` already depends on `infrastructure-controllers`; keep `wait: true` on infra layers.

CSI details:

- Helm release name: `hcloud-csi`
- Namespace: `kube-system`
- Chart repo: `https://charts.hetzner.cloud`
- Chart: `hcloud-csi`
- Values:
  - reference existing Secret `hcloud` key `token` via `controller.hcloudToken.existingSecret.name/key`
  - set `controller.hcloudVolumeDefaultLocation: nbg1` unless the current chart docs recommend auto-detection for this topology
- Expect StorageClass `hcloud-volumes` from the chart unless the chart docs have changed. If adding a custom StorageClass under `flux/infrastructure/configs`, make it explicit and avoid duplicate defaults.

Credential ownership:

- Ansible creates `kube-system/hcloud`; Flux references it.
- Do not place the hcloud token in unencrypted Git manifests.
- If the team later wants Flux to own this Secret, plan a separate SOPS-encrypted Kubernetes Secret migration with a no-downtime handoff.

## Ordered implementation steps

1. **Refresh targeted context and docs** — owner: `repo-fast-context` + `web-fast-context`
   - Re-read the exact current files listed in this plan before editing.
   - Re-check official docs/chart values and chart versions.
   - Confirm no newer ADR or plan has superseded this install boundary.

2. **Resolve live-session blockers** — owner: live orchestrator + user
   - Confirm the CSI boundary: Flux-first as recommended, or pre-Flux override.
   - Fill `flux_repo_url` and decide Flux auth method (SSH deploy key/agent vs HTTPS credentials).
   - Confirm whether this implementation should update existing ADRs immediately after code review or wait for a separate ADR-only pass.

3. **Implement Talos/provider variables and bootstrap role** — owner: `ansible-implementor`
   - Add conditional `cluster.externalCloudProvider.enabled: true` to `common.yaml.j2` as shown above.
   - Add default/override vars for `cluster_external_cloud_provider_enabled` and `flux_bootstrap_pre_flux_components`.
   - Add pinned `kubernetes.core` to Ansible requirements if using its modules.
   - Implement `flux_bootstrap` tasks for preflight, hcloud Secret, Cilium Helm install, hcloud CCM Helm install, readiness checks, and Flux bootstrap.
   - Ensure every task is idempotent, uses FQCNs, has `no_log: true` on token handling, and avoids printing Secret values.

4. **Implement Flux controller manifests** — owner: `k8s-implementor`
   - Add Cilium, hcloud CCM, and hcloud CSI HelmRepository/HelmRelease resources under `flux/infrastructure/controllers/`.
   - Pin chart versions.
   - Match Cilium and CCM values exactly with the Ansible pre-Flux installs for clean adoption.
   - Add HelmRelease `dependsOn` chain: Cilium → hcloud CCM → hcloud CSI.
   - Keep the existing `infrastructure-controllers` → `infrastructure-configs` → `apps` Kustomization ordering, with `interval`, `prune`, and `wait` preserved.

5. **Update user-facing bootstrap docs** — owner: `docs-writer`
   - Update the bootstrap runbook/docs to describe the real `bootstrap-flux.yml` flow: Cilium, hcloud Secret, hcloud CCM, Flux bootstrap, Flux adoption, CSI via Flux.
   - Include the non-obvious taint behavior from external cloud provider mode.
   - Include live-cluster and cost warnings for CSI PVC tests.
   - Keep `CLAUDE.md` implementation status honest if components become implemented.

6. **ADR follow-up after decision confirmation** — owner: `adr-writer`
   - Write a kube1-specific ADR under `docs/kube1/adrs/` for deploying hcloud CCM and enabling Talos external cloud provider on kube1, or update an existing kube1 ADR if the user prefers.
   - Update/cross-reference the existing cluster-bootstrap ADR inconsistency: ADR-005 says CCM remains outside Flux, while ADR-010 says Flux adopts it. The intended decision for this plan is **pre-Flux install, Flux adoption/ownership**.
   - Do not finalize this silently; get user confirmation of the install-boundary decision first.

7. **Review Ansible lane** — owner: `ansible-reviewer`
   - Audit idempotency, FQCNs, variable namespacing, check-mode behavior, no-log Secret handling, and live-cluster safety.
   - Run applicable Ansible validation from the validation section.
   - Confirm no task can leak `hcloud_token` in command args, logs, diffs, or failure messages.

8. **Review Flux/Kubernetes lane** — owner: `k8s-reviewer`
   - Audit HelmRelease versions, adoption compatibility, dependency ordering, wait/prune/interval conventions, and Secret references.
   - Render/lint Kustomize/Flux manifests.
   - Confirm CSI does not race CCM/providerID initialization.

9. **Fix review findings** — owner: matching implementor
   - Route Ansible findings back to `ansible-implementor`.
   - Route Flux/Kubernetes findings back to `k8s-implementor`.
   - Re-review until both lanes pass.

10. **Closing report** — owner: `report-writer`
    - Write `.ai/reports/2026-06-22-cilium-ccm-csi.md` after implementation and review.
    - Record what changed, what was validated, what the user still needs to run live, and any ADR/docs follow-ups.

## Validation required before handoff

### Static/local validation by implementors/reviewers

From `infra/ansible/`:

```bash
ansible-lint
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-flux.yml --syntax-check
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check
```

Run `--check --diff` only where it does not contact or mutate the live cluster unexpectedly. This playbook is intentionally a live Kubernetes bootstrap path; if no safe kubeconfig/test cluster is available, document why full check-mode validation was not run and rely on syntax/lint plus reviewer audit. If a safe kubeconfig is available and the user approves live API access, run:

```bash
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-flux.yml --check --diff
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --check --diff
```

From repo root for Flux/Kubernetes manifests:

```bash
kustomize build flux/clusters/kube1
kustomize build flux/infrastructure/controllers
kustomize build flux/infrastructure/configs
flux check --pre
```

If Helm templating validation is available, template the pinned charts with the planned values before handoff.

### Live validation for the user to run after approval

The user, not agents, runs live cluster operations unless explicitly approved.

1. Apply Talos machine config change:
   ```bash
   cd infra/ansible
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml
   ```
   Watch for Talos reporting a reboot-required change. If a reboot is required, stop and plan a rolling Talos-safe sequence.

2. Run pre-Flux bootstrap and Flux bootstrap:
   ```bash
   cd infra/ansible
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml
   ```

3. Validate Cilium:
   ```bash
   kubectl -n kube-system rollout status ds/cilium
   kubectl -n kube-system exec ds/cilium -- cilium-dbg status
   kubectl get nodes -o wide
   ```
   Expected: Cilium is Ready, kube-proxy replacement is enabled, nodes become/return Ready.

4. Validate hcloud CCM:
   ```bash
   kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" providerID="}{.spec.providerID}{" taints="}{.spec.taints}{"\n"}{end}'
   kubectl get nodes --show-labels
   ```
   Expected: no `node.cloudprovider.kubernetes.io/uninitialized` taint remains, each node has `providerID=hcloud://...`, and topology/provider labels are present.

5. Validate Flux adoption:
   ```bash
   flux get kustomizations -A
   flux get helmreleases -A
   helm -n kube-system list
   ```
   Expected: Cilium and hcloud CCM HelmReleases become Ready without uninstall/reinstall churn; hcloud CSI is installed by Flux and Ready.

6. Validate CSI provisioning only after accepting cost:
   - Create a temporary PVC and pod using `hcloud-volumes`.
   - Confirm PVC binds and a Hetzner Cloud Volume is created/attached.
   - Delete the test pod/PVC immediately after validation.

This CSI test creates a billable Hetzner Cloud Volume while the PVC exists. Check live Hetzner pricing before quoting cost or leaving test volumes around.

## Risks, side effects, cost-incurring operations, and live-cluster boundaries

- **Bootstrap taint risk:** Once `cluster.externalCloudProvider.enabled: true` is applied, nodes receive the uninitialized taint until hcloud CCM runs. If CCM install fails, new workload scheduling can be blocked. Keep CCM pre-Flux and validate taint removal before workload rollout.
- **CNI race risk:** Flux controllers cannot rely on pod networking until Cilium is installed and Ready. Do not run `flux bootstrap` before Cilium is ready.
- **Adoption risk:** Flux adoption depends on exact release names/namespaces and compatible chart names/values. Changing `releaseName` or `targetNamespace` later can uninstall/reinstall controllers.
- **Secret leakage risk:** `hcloud_token` must stay in SOPS/Ansible memory and Kubernetes Secret only. Use `no_log: true`; do not write decrypted tokens to files, command args, diffs, or Git manifests.
- **Provider boundary risk:** Do not enable external cloud provider for `manual` inventory unless a matching CCM exists; otherwise nodes can remain permanently tainted.
- **API side effects:** `flux bootstrap git` writes to the Git repository and installs/updates Flux controllers. It is idempotent but not side-effect-free.
- **Hetzner API side effects:** hcloud CCM and CSI use the hcloud API. Controller install itself should not create billable resources; CSI PVC validation creates billable Cloud Volumes.
- **Cost boundary:** Cilium/CCM/Flux do not directly create new Hetzner billable resources. CSI validation and any real PVCs do.
- **Live-cluster boundary:** Agents must not run provisioning, bootstrap, `kubectl`, `flux`, `hcloud`, or `talosctl` against the real cluster without explicit user approval. The user runs the live playbooks.

## ADR needs

- **ADR required after confirmation:** kube1-specific ADR under `docs/kube1/adrs/` for hcloud CCM + Talos external cloud provider bootstrap contract.
- **ADR update/cross-reference required:** resolve the existing `cluster-bootstrap/adrs/005-cloud-controller-manager-pattern.md` vs ADR-010 inconsistency about whether CCM remains outside Flux or is adopted by Flux. This plan chooses adoption.
- **CLAUDE.md architecture table update:** after implementation, update `Hetzner CCM` from `Not deployed — no LoadBalancer-type services needed yet` to deployed for node lifecycle/providerID/taint clearing, with route/LB controllers not used unless values say otherwise.
- **No new CSI ADR is needed** if CSI remains Flux-managed as in existing kube1 ADR-005. If the user overrides and makes CSI pre-Flux, update the CSI/hand-off ADRs.

## Open questions before implementation

1. Confirm CSI install boundary: accept recommended **Flux-managed from first reconciliation**, or override to pre-Flux + Flux adoption.
2. Provide/set `flux_repo_url` and Flux authentication method for `flux bootstrap git`.
3. Confirm whether `CLAUDE.md` should be updated in the same implementation slice or only after live validation succeeds.
4. Confirm whether to write the hcloud CCM ADR in the same workstream after implementation review, or as a separate ADR-only follow-up.
