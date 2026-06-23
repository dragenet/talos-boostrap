# Pre-Flux CNI and CCM bootstrap — closing report

**Date:** 2026-06-23
**Plan:** `.ai/plans/2026-06-23-pre-flux-cni-ccm-bootstrap.md`
**Type:** Feature implementation — Cilium CNI and hcloud CCM pre-Flux install
with Flux adoption, plus the config schema migration that promoted CNI/CCM
from the optional `features` catalog to top-level core config keys.

The render/config-compiler refactor (`.ai/reports/2026-06-22-refactor-render-architecture.md`)
built the pipeline; this slice fed it the first two real components and
graduated CNI / CCM out of `features.*`.

## What changed

### Config schema migration (ADR-002, ADR-005, ADR-010, ADR-011, ADR-012)

- `config/defaults/cluster.yaml` and `config/overrides/cluster.yaml` —
  `features.cni` and `features.hcloudCcm` removed. Replaced by top-level
  core keys:
  - `cni.name` (`cilium` | `flannel` | `none`; defaults to `cilium`).
  - `cloud_controller_manager.enabled` (tri-state: `auto` | `true` | `false`;
    default `auto`).
- `config_render` validation (`roles/config_render/tasks/validate.yml`) —
  rejects stale `features.cni` / `features.hcloudCcm` keys in overrides
  with a clear migration message; rejects the deprecated `talos.network.vip`
  in favour of `cluster.vip` (single source of truth).
- `config_render` compute (`roles/config_render/tasks/compute.yml`) —
  resolves the `auto` tri-state per `provider.name`: `hcloud` resolves
  `auto` → `true` (CCM available), `manual` resolves `auto` → `false`
  (no CCM). Computed keys (`talos_extensions`,
  `bootstrap_pre_flux_components`, `cluster_endpoint`,
  `cluster_external_cloud_provider_enabled`) are rejected as direct
  overrides in `validate.yml`.
- `bootstrap_pre_flux_components` is now computed from
  `cni.name` + `cloud_controller_manager.enabled` + `provider.name`:
  hcloud + cilium → `[cilium, hcloud-ccm]`; manual + cilium → `[cilium]`;
  cni=none → `[]`.
- `cluster_external_cloud_provider_enabled` (gate on the all-node
  Talos `10-cloud-provider.yaml.j2` patch) is now derived from the
  pre-Flux list, not from the raw config value — preventing the
  "uninitialized taint with no CCM to clear it" failure mode.

### Generated outputs (now hcloud-CCM-on by default)

- `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.yml` —
  `bootstrap_pre_flux_components: [cilium, hcloud-ccm]` (was `[cilium]`).
- `infra/talos/patches/generated/all/10-cloud-provider.yaml.j2` — newly
  generated (hcloud CCM now selected), sets `cluster.externalCloudProvider.enabled: true`.
- `flux/infrastructure/controllers/base/kustomization.yaml` — render-owned
  base now references `../_components/cilium` and `../_components/providers/hcloud/ccm`
  (was `resources: []`).
- `infra/ansible/roles/config_render/templates/flux/base/kustomization.yaml.j2` —
  new template backing the base renderer. Iterates the computed list
  (`cilium` → `../_components/cilium`; `hcloud-ccm` → `../_components/providers/hcloud/ccm`).
- Manual inventory (`providers/manual/group_vars/all/bootstrap.yml.j2`,
  rendered `bootstrap.yml`) — stays at `[cilium]`, no CCM, no
  `externalCloudProvider` patch.

### `flux_bootstrap` role — real implementation (was a comment-only stub)

- `infra/ansible/playbooks/bootstrap-flux.yml` — playbook header
  rewritten to document the full pre-Flux → Flux flow.
- `infra/ansible/roles/flux_bootstrap/tasks/main.yml` — orchestrator
  running `preflight → namespaces → cilium → hcloud_ccm → flux` in order.
  Each component step is gated on its presence in
  `bootstrap_pre_flux_components`; the Flux step is gated on
  `flux_repo_url` being non-empty.
- `tasks/preflight.yml` (new) — checks `kubectl` / `helm` / `flux`
  CLI tools, asserts `flux_repo_url` non-empty, asserts `hcloud_token`
  set when CCM is needed, checks the kubeconfig exists and the cluster
  API is reachable.
- `tasks/namespaces.yml` (new) — idempotent `kubernetes.core.k8s`
  creation of `cilium-system` and `hcloud-ccm-system`.
- `tasks/cilium.yml` (new) — `kubernetes.core.helm` install/upgrade of
  the `cilium` release into `cilium-system` with the full Talos-required
  values (KubePrism endpoint, `cgroup.autoMount.enabled: false`, capability
  set without `SYS_MODULE`, `bpf.hostLegacyRouting: true`, full
  `operator.tolerations` including the uninitialized-taint entry to
  avoid Cilium issue #41921 deadlock). Waits on
  `daemonset/cilium` and `deployment/cilium-operator` rollout; tolerates
  CoreDNS-rollout failure (best-effort) and notes that DNS issues
  surface in subsequent steps.
- `tasks/hcloud_ccm.yml` (new) — `kubernetes.core.k8s` Secret
  `hcloud-ccm-system/hcloud` (key `token`) from `hcloud_token` with
  `no_log: true`; `kubernetes.core.helm` install/upgrade of the
  `hcloud-cloud-controller-manager` release with
  `networking.enabled: false`. Post-install contract checks (retries
  12×10 s): no node carries the
  `node.cloudprovider.kubernetes.io/uninitialized` taint; every
  `spec.providerID` starts with `hcloud://`.
- `tasks/flux.yml` (new) — idempotent `flux bootstrap git` guarded by
  a `kubectl get deployment helm-controller -n flux-system` pre-check,
  using the rendered `flux_repo_url` / `flux_repo_branch` / `flux_path`.
  The pre-check prevents re-runs from reporting `changed` (Flux prints
  "all components are healthy" on both first-run and re-run — there
  is no native "I did nothing" signal). Limitation called out in
  comments: a partial bootstrap (controllers installed but deploy key
  not pushed) is not auto-recoverable — delete `flux-system` and re-run.
- `defaults/main.yml` (new) — all role-local `flux_bootstrap_*` defaults
  (chart versions, namespaces, release names, values, timeouts) form
  the **Flux adoption contract** with the matching `_components/`
  HelmReleases. Pinned: Cilium chart `1.19.5`; hcloud CCM chart
  `1.33.0` (avoids the `<= v1.30.0` July-2026 Hetzner
  `server.datacenter` deprecation risk).
- `infra/ansible/requirements.yml` — pins `kubernetes.core: 6.4.0`
  (the k8s and helm modules). `hetzner.hcloud: 6.9.0` and
  `community.sops: 2.3.0` unchanged.

### Flux catalog entries (template-owned, Flux-adoption-side of the contract)

- `flux/infrastructure/controllers/_components/cilium/` (new) —
  `namespace.yaml` (`cilium-system`), `helmrepository.yaml`
  (`https://helm.cilium.io`), `helmrelease.yaml` (chart `1.19.5`,
  `kubeProxyReplacement: true`, `routingMode: tunnel`, `l7Proxy: false`,
  `gatewayAPI.enabled: false`, `k8sServiceHost: localhost`,
  `k8sServicePort: "7445"` for Talos KubePrism, `ipam.mode: kubernetes`,
  `cgroup.autoMount.enabled: false`, `cgroup.hostRoot: /sys/fs/cgroup`,
  full capability list, `bpf.hostLegacyRouting: true`, full
  `operator.tolerations` with the uninitialized-taint entry),
  `kustomization.yaml` (`namespace → helmrepository → helmrelease`).
- `flux/infrastructure/controllers/_components/providers/hcloud/ccm/` (new) —
  `namespace.yaml` (`hcloud-ccm-system`), `helmrepository.yaml`
  (`https://charts.hetzner.cloud`), `helmrelease.yaml` (chart `1.33.0`,
  `networking.enabled: false`, `dependsOn: [{name: cilium, namespace: cilium-system}]`),
  `kustomization.yaml`.
- Token ownership contract: the CCM HelmRelease references the
  `hcloud` Secret by chart-default `envFrom`; Flux does **not** own
  a Secret manifest, so no plaintext token ever reaches Git.

### ADR and CLAUDE.md status updates

- `docs/cluster-bootstrap/adrs/002-cilium-cni.md` — extended to cover
  the `cni.name` schema (allowed values `flannel` | `cilium`), the
  pre-Flux install / Flux adoption pattern, and the
  `cilium-system` namespace choice. ADR is now core-config-agnostic
  (the original "default" framing is replaced with "selection and
  configuration"). Implementation status callout appended.
- `docs/cluster-bootstrap/adrs/005-cloud-controller-manager-pattern.md` —
  rewritten to document the `auto` / `true` / `false` tri-state and
  the provider-resolution rule (hcloud → enabled; manual → disabled).
  Status changed from "Superseded by ADR-010" to **Accepted**.
- `docs/cluster-bootstrap/adrs/010-ansible-flux-provider-handoff.md` —
  added the core-config / features separation to the "What goes where"
  table; clarified that pre-Flux components use a dedicated
  namespace and are adopted by Flux.
- `docs/cluster-bootstrap/adrs/012-feature-catalog.md` — added a
  **core bootstrap** row to the taxonomy table; removed CNI and CCM
  from the Talos-coupled / provider-gated rows (they live in their
  own top-level keys now).
- `CLAUDE.md` — Implementation status block updated: "Working — pre-Flux
  + Flux (Cilium, hcloud CCM, Flux adoption)" replaces the previous
  stub note. Pre-Flux bootstrap section expanded to include the
  namespace / taint / Secret contracts and the full live-validation
  command list. Repo-structure tree updated to call out the new
  `_components/cilium` and `_components/providers/hcloud/ccm` entries.
- `.opencode/agents/k8s-implementor.md` — one-character typo fix
  (`deppseek-v4-pro` → `deepseek-v4-pro`); no content change. (Not
  in scope for this slice; included for accuracy.)

## Key files

### New
- `infra/ansible/roles/flux_bootstrap/tasks/{preflight,namespaces,cilium,hcloud_ccm,flux}.yml`
- `infra/ansible/roles/flux_bootstrap/defaults/main.yml`
- `infra/ansible/roles/config_render/templates/flux/base/kustomization.yaml.j2`
- `flux/infrastructure/controllers/_components/cilium/{namespace,helmrepository,helmrelease,kustomization}.yaml`
- `flux/infrastructure/controllers/_components/providers/hcloud/ccm/{namespace,helmrepository,helmrelease,kustomization}.yaml`
- `infra/talos/patches/generated/all/10-cloud-provider.yaml.j2`

### Modified (config + render)
- `config/defaults/cluster.yaml`, `config/overrides/cluster.yaml`
- `infra/ansible/roles/config_render/{defaults/main,tasks/compute,tasks/validate,tasks/render_flux,tasks/render_talos,tasks/check,tasks/main}.yml`
- `infra/ansible/roles/config_render/templates/group_vars/hcloud/hcloud.yml.j2`
- `infra/ansible/roles/config_render/templates/group_vars/manual/bootstrap.yml.j2`
- `infra/ansible/roles/config_render/templates/talos_patches/{all/10-cloud-provider,controlplane/10-cni-proxy}.yaml.j2`
- `infra/ansible/roles/flux_bootstrap/tasks/main.yml`
- `infra/ansible/playbooks/bootstrap-flux.yml`
- `infra/ansible/requirements.yml`
- `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.yml` (rendered)
- `infra/talos/patches/generated/controlplane/10-cni-proxy.yaml.j2` (rendered, comment-only edit)
- `flux/infrastructure/controllers/base/kustomization.yaml` (rendered)

### Modified (docs / ADRs)
- `CLAUDE.md` (Implementation status, repo tree, Pre-Flux section)
- `docs/cluster-bootstrap/adrs/002-cilium-cni.md`
- `docs/cluster-bootstrap/adrs/005-cloud-controller-manager-pattern.md`
- `docs/cluster-bootstrap/adrs/010-ansible-flux-provider-handoff.md`
- `docs/cluster-bootstrap/adrs/012-feature-catalog.md`
- `docs/upgrades.md`, `docs/node-roles.md`, `docs/adding-a-provider.md`
  (render-driven workflow notes — no architecture change)
- `.opencode/agents/k8s-implementor.md` (typo fix only)

## What was validated

All checks local and read-only; no live cluster contact, no provider-costing
commands, no `hcloud` / `talosctl` / `kubectl` / live `helm` / `flux`
operations.

- **`ansible-lint .`** from `infra/ansible/` — PASS
  (`Passed: 0 failure(s), 0 warning(s)`, production profile).
- **`ansible-playbook playbooks/bootstrap-flux.yml --syntax-check`** — PASS.
- **`ansible-playbook playbooks/render-config.yml --syntax-check`** — PASS.
- **`ansible-playbook playbooks/render-config.yml -e config_render_check=true`** —
  PASS: "No render drift detected. Committed generated files match render
  output." (two consecutive runs produce no second-run diff — deterministic).
- **`kustomize build flux/infrastructure/controllers/`** — PASS: emits
  `cilium` and `hcloud-ccm` namespaces, `HelmRepository` (Cilium and
  hcloud), and `HelmRelease` resources for both with all values intact.
- **`kustomize build flux/infrastructure/controllers/_components/cilium`**
  and **`_components/providers/hcloud/ccm`** — PASS individually.
- **Render output spot-checks:**
  - `hcloud.yml` `bootstrap_pre_flux_components: [cilium, hcloud-ccm]`.
  - `all/10-cloud-provider.yaml.j2` is now present and emits
    `cluster.externalCloudProvider.enabled: true`.
  - `base/kustomization.yaml` references both `_components/cilium` and
    `_components/providers/hcloud/ccm`.
- **Adoption-contract parity:** `flux_bootstrap_cilium_*` defaults
  (release name, namespace, chart version, full values) match
  `_components/cilium/helmrelease.yaml` exactly; same check passes
  for hcloud CCM.
- **Reviewer passes:** the implementor ran both the Ansible and K8s
  reviewer subagents; both PASSED. Notes from the review:
  - Ansible: FQCN everywhere, no_log on the Secret creation, no
    `hcloud_token` in command args or rendered files, idempotent
    `kubernetes.core.k8s` and `kubernetes.core.helm` use, computed-key
    guard via `validate.yml`, no live-cluster contact.
  - K8s: chart versions pinned (`1.19.5` for Cilium,
    `1.33.0` for hcloud CCM — `1.30.1+` to avoid the July 2026
    Hetzner `server.datacenter` deprecation), release names / namespaces
    / values match the Ansible defaults byte-for-byte, `dependsOn`
    wiring in the CCM HelmRelease, no plaintext Secret manifest under
    flux/, no `:latest` / floating tags.

## Live commands the user still needs to run

### Before `bootstrap-flux.yml`

`config/overrides/cluster.yaml` has `flux.repoUrl: ""`. The pre-Flux
Cilium and CCM steps will install and become Ready, but the final
`flux bootstrap git` step in `flux_bootstrap/tasks/flux.yml` is gated
on a non-empty `flux_repo_url` and will fail at the assertion. Set
`flux.repoUrl` in `config/overrides/cluster.yaml` (and configure the
matching authentication — SSH deploy key or HTTPS credentials — on the
git host), then re-render:

```bash
cd infra/ansible
ansible-playbook playbooks/render-config.yml
```

The Cilium and CCM pre-Flux steps do not block on `flux_repo_url`;
they will run even when it is empty, which is intentional — it lets
the user validate the pre-Flux layer in isolation before configuring
the git remote.

### After Talos is up and `flux.repoUrl` is set

```bash
cd infra/ansible
ansible-galaxy collection install -r requirements.yml   # one-time, pins kubernetes.core
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml
```

### Live validation (run in user's own terminal — agents must not run)

```bash
# Cilium Ready in cilium-system
kubectl -n cilium-system rollout status daemonset/cilium
kubectl -n cilium-system get pods -o wide

# hcloud CCM cleared the uninitialized taint and set providerID
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" providerID="}{.spec.providerID}{" taints="}{.spec.taints}{"\n"}{end}'
# Expected: no "node.cloudprovider.kubernetes.io/uninitialized" in any
# taints line; every providerID starts with "hcloud://".

# Flux controllers Ready, infra-controllers kustomization Ready
kubectl -n flux-system get pods
flux get kustomizations -A
flux get helmreleases -A
# Expected: cilium and hcloud-cloud-controller-manager are listed as
# Ready HelmReleases with no install/uninstall churn (adoption was clean).

# Helm CLI cross-check: the same releases exist with the same names
helm -n cilium-system list
helm -n hcloud-ccm-system list
```

If any check fails, do **not** proceed to install workloads — the
CCM is a hard prerequisite for every other pod scheduling.

## Remaining follow-ups / deferred items

- **`flux.repoUrl` is still empty.** Live Flux bootstrap cannot succeed
  until the user fills `config/overrides/cluster.yaml` `flux.repoUrl`
  and re-renders. Same authentication-method precondition
  (deploy key or HTTPS credentials) as before.
- **Hetzner CSI is not installed.** `features.hcloudCsi: false` and no
  catalog entry. StorageClasses for cloud volumes are not yet defined.
  Out of scope for this slice; will be its own follow-up plan.
- **`cert-manager`, `OpenEBS`, `ingress` (envoy-gateway / traefik) not
  installed.** `features.certManager`, `features.openebs`, and
  `features.ingress` all default to off / `none`; no catalog entries
  exist. Will follow the same catalog / base / overlay pattern when
  built.
- **`flux/infrastructure/configs/` and `flux/apps/` are still empty
  `resources: []` placeholders.** No `ClusterIssuers`, `StorageClasses`,
  or workload Helm releases yet.
- **`talosctl` deviceSelector probe (ADR-013) is still deferred.**
  The current `interface: eth1` form in `config/overrides/cluster.yaml`
  remains. The user runs the probe after the cluster is bootstrapped
  to feed `ansible-implementor` for a real `deviceSelector`.
- **Manual provider live validation is unverified.** The manual code
  paths (`bootstrap_pre_flux_components: [cilium]`, no CCM, no
  `externalCloudProvider` patch) are wired through the compiler and
  validated by `render-config.yml --check` / `kustomize build`, but
  have not been exercised against a live cluster.
- **Partial-bootstrap recovery is manual.** If `flux bootstrap git`
  fails after installing the controllers but before pushing the deploy
  key, the idempotency gate in `flux_bootstrap/tasks/flux.yml` will
  skip re-running. Recovery: `kubectl delete namespace flux-system`
  and re-run the playbook. This is called out in a comment in the
  task but is not automated.
- **CI for render drift is not wired.** Render check remains a
  local-only validation command (`-e config_render_check=true`),
  matching the prior slice's deferral.
- **Live cluster state is unverified.** The "Working — pre-Flux +
  Flux" entry in `CLAUDE.md` Implementation status is honest: every
  item in this slice has been reviewed, but the user has not yet run
  `bootstrap-flux.yml` against a live cluster, so "working" means
  "the code is built and passes review/validation", not "we have
  observed it running".

## Risks worth re-flagging at handoff

- **External cloud provider taint.** The Talos all-node patch now
  sets `cluster.externalCloudProvider.enabled: true` whenever hcloud
  CCM is in the pre-Flux list. Talos will taint every node
  `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` until
  the CCM clears it. If the CCM install is skipped (e.g. user flips
  `cloud_controller_manager.enabled: false` without re-rendering
  the Talos layer), nodes stay tainted forever and no workloads
  schedule. The render guards this by deriving the patch from the
  pre-Flux list — flipping CCM off also disables the patch — but a
  live-cluster user mutating the Talos layer out-of-band would
  re-introduce it. Don't.
- **Helm adoption sensitivity.** Ansible and Flux must use the same
  release name, namespace, chart version, and values. A drift
  (renaming `cilium` to `cilium-cn` in one place, for instance)
  causes Helm to uninstall and reinstall — which can drop in-flight
  pod networking. The defaults on both sides are pinned in
  `flux_bootstrap/defaults/main.yml` and
  `_components/<name>/helmrelease.yaml`; never edit one without
  the other.
- **Secret boundary.** `hcloud_token` is provider API access. It is
  SOPS-decrypted at runtime, used only by the `kubernetes.core.k8s`
  Secret creation task (`no_log: true`), and never written into a
  generated file or Git manifest. The Flux HelmRelease references
  the Secret by chart default `envFrom`; no plaintext Secret
  manifest exists under `flux/`.
- **Flux bootstrap side effects.** `flux bootstrap git` modifies the
  live cluster and can commit/push Flux manifests to the configured
  Git remote. Make sure `flux.repoUrl` points at the intended repo
  before running.

## Artifacts

- Plan: `.ai/plans/2026-06-23-pre-flux-cni-ccm-bootstrap.md`
- Report: `.ai/reports/2026-06-23-pre-flux-cni-ccm-bootstrap.md`
  (this file)
- ADRs updated: `docs/cluster-bootstrap/adrs/002-cilium-cni.md`,
  `005-cloud-controller-manager-pattern.md`,
  `010-ansible-flux-provider-handoff.md`,
  `012-feature-catalog.md`
- Status updated: `CLAUDE.md` Implementation status block, repo
  structure tree, Pre-Flux section
