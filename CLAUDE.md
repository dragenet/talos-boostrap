# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Infrastructure-as-code for `kube1`: a 3-node Talos Linux Kubernetes cluster on Hetzner Cloud. All nodes are hybrid (control-plane + worker). Everything — from VM provisioning to workload deployment — is code-driven with no manual steps.

> **Always fetch Hetzner server specs and pricing live from hetzner.com before quoting or configuring anything. Training data is stale — Hetzner raised prices ~37% in April 2026.**

## How we work together

This is a learning-focused homelab. Optimise for my understanding, not just a working cluster.

- **Teaching mode is on.** When you introduce a new tool, flag, resource kind, or pattern, explain in a sentence or two what it does, the problem it solves, and the alternative we're *not* using and why. Lead with the conceptual model, then the concrete command. Don't assume prior Kubernetes / Talos / Hetzner / Ansible knowledge — but keep it tight and inline, not walls of text.
- **Explain the *why*, not just the *how*.** Tie each infra decision back to its trade-off: HA, latency, cost, security, or blast radius. Every row in the Architecture table has a reason — surface it.
- **Flag non-obvious behaviour before I run anything** — idempotency, side effects, destructive flags, cost-incurring API calls, anything that recreates servers or rotates secrets.
- **Docs before code.** For any external tool (Talos, Hetzner, Flux, Cilium, cert-manager, the `hetzner.hcloud` collection) read the official docs/README before proposing config or answering — training knowledge is stale. Canonical sources: talos.dev & docs.siderolabs.com, docs.hetzner.com, fluxcd.io, docs.cilium.io, docs.ansible.com.
- **You validate; I run.** I run `ansible-playbook` against the real cluster in my own terminal — don't auto-run them. Before handing a new or edited playbook/role back to me, run `ansible-lint` then `ansible-playbook … --check --diff` yourself and report the result.

## Subagents

This repo ships specialized subagents in `.claude/agents/`. Use them deliberately — don't spawn for trivial one-liners, but do route real implementation/review work through them. Standard flow: **plan → implement → review → report.**

| Subagent | Role | Model / effort |
|---|---|---|
| `task-planner` | Breaks a goal into ordered, dependency-aware steps and assigns each to the right subagent. **Call first** for any non-trivial or multi-file/multi-layer task. Writes the plan to `.ai/plans/`; never edits repo code. | opus / high |
| `web-fast-context` | Read-only, low-cost official-docs/external-reference fetcher. Use in parallel with repo context when docs and repo facts are independent. | haiku / low |
| `ansible-implementor` | Writes/edits playbooks, roles, inventories, group_vars under `infra/ansible/`. | sonnet / low |
| `k8s-implementor` | Writes/edits K8s manifests, Kustomize, Helm, and Flux resources under `flux/`. | sonnet / low |
| `ansible-reviewer` | Read-only audit of Ansible changes (idempotency, FQCN, namespacing, secrets); runs `ansible-lint` + `--check`. | opus / medium |
| `k8s-reviewer` | Read-only audit of K8s/Flux changes (versions pinned, dependsOn/wait, CNI ordering, correctness). | opus / medium |
| `docs-writer` | Writes/updates user-facing docs under `docs/` (and `README.md`) — runbooks, how-to guides, operational procedures. **Not** ADRs/rationale, **not** `.ai/` artifacts. | sonnet / low |
| `adr-writer` | Writes/updates Architecture Decision Records under `docs/**/adrs/` — the durable *why* behind a choice (Nygard: Context / Decision / Consequences). For real architectural decisions with trade-offs and rejected alternatives. **Not** runbooks, **not** `.ai/` artifacts. | sonnet / low |
| `report-writer` | Writes a concise closing report to `.ai/reports/` after work is done — what changed, what was validated, follow-ups. **Call last.** Summarizer only; doesn't investigate or edit. | haiku / low |

How to use them:
- **Plan before implementing.** For anything beyond a trivial edit, dispatch `task-planner` first. It writes a dated plan to `.ai/plans/<date>-<slug>.md` and returns the path + a summary; dispatch the implementor(s) it names.
- **Use graphify before raw repo search.** For codebase questions, query graphify first. Use LSP and targeted reads only after graphify has narrowed the area. Use `web-fast-context` for external docs/facts.
- **Implementors lint before handoff — linting is their quality gateway.** Ansible implementor runs `ansible-lint` (+ `--check --diff` where feasible); the K8s implementor renders/lints (`kustomize build`, `flux build`, `helm lint/template`, `kubeconform`/`yamllint`). Never hand off code that fails its linter.
- **Review after implementing.** Send Ansible changes to `ansible-reviewer` and Flux/K8s changes to `k8s-reviewer`. Loop fixes back to the matching implementor until the reviewer approves.
- **Document user-facing changes.** When a workflow or feature lands that someone has to operate, dispatch `docs-writer` to add/update the runbook or guide under `docs/`. It writes operational docs only — never ADRs/design rationale and never `.ai/` artifacts.
- **Record architectural decisions.** When a real architectural choice is made or changed (a tool/pattern/topology decision with trade-offs and rejected alternatives), dispatch `adr-writer` to author the ADR under `docs/**/adrs/` — provider-agnostic choices go in `docs/cluster-bootstrap/adrs/`, kube1-specific ones in `docs/kube1/adrs/`. The `CLAUDE.md` Architecture table stays the one-line index the user owns; the ADR holds the full rationale.
- **Report when done.** For multi-step tasks, dispatch `report-writer` last; it writes `.ai/reports/<date>-<slug>.md` (mirroring the plan slug) recording what changed, what was validated, and open items. Skip it for trivial one-off edits.
- Implementors do not self-review and do not cross domains (Ansible vs K8s). Reviewers never edit; the planner and report-writer only write their own artifacts under `.ai/`; docs-writer stays in `docs/`/`README.md` and adr-writer stays in `docs/**/adrs/` — none of them touch repo code outside their lane.

**`.ai/` is the durable scratch/record dir** (committed, not gitignored): `.ai/plans/` holds planner output, `.ai/reports/` holds closing reports, and ad-hoc reviews live at `.ai/` root (e.g. `.ai/review-2026-06-10.md`). All use a dated `YYYY-MM-DD-<slug>.md` naming convention.

## Architecture

| Concern | Decision |
|---|---|
| OS | Talos Linux (immutable, API-driven, no SSH) |
| Nodes | 3× CX33, all in Nuremberg (`nbg1`), single DC |
| Node role | Three roles: `controlplane` (etcd only, workloads tainted off), `worker` (workloads only), `hybrid` (both). Initial 3 nodes are `hybrid`. Full names everywhere — in inventory groups, Hetzner labels, Kubernetes `node-role.kubernetes.io/*` labels. Short codes (`cp`/`wk`/`hb`) used only in generated hostnames (e.g. `kube1-hb1`). |
| Inter-node network | Hetzner private network (L3 routed within `eu-central`) |
| K8s API endpoint (internal) | Talos native VIP — ARP-based floating IP across the 3 nodes, no LB needed |
| K8s API endpoint (external) | Cloudflare Tunnel — zero open ports, zero-trust `kubectl` access |
| GitOps | FluxCD |
| CNI | Cilium, replaces kube-proxy |
| Block storage | Hetzner CSI driver — auto-provisions Hetzner Cloud Volumes as PVCs |
| Local storage | OpenEBS LocalLV LVM — low-latency local storage for latency-sensitive workloads |
| TLS | cert-manager |
| Hetzner CCM | Pre-Flux Ansible install in `hcloud-ccm-system`; Flux adopts. With `networking.enabled: false` — no route or `LoadBalancer` controllers (Cilium owns routing). Clears the `node.cloudprovider.kubernetes.io/uninitialized` taint and sets `spec.providerID: hcloud://…` on every node. |
| Ingress | TBD |

**Why single DC:** Talos native VIP is ARP-based (L2). Hetzner private networks are L3-routed — ARP does not propagate across DC boundaries even within `eu-central`. Multi-DC would require a Hetzner LB for the API endpoint.

## Repository structure

```
kube1/
├── config/                       # Render-compiler inputs (ADR-011, ADR-016)
│   ├── defaults/                 # TEMPLATE-OWNED — provider-agnostic defaults
│   │   └── cluster.yaml
│   ├── talos/                    # TEMPLATE-OWNED — shared Talos-shaped patch base + components
│   │   ├── base/                 #   cluster-agnostic Talos machine config patches (all, controlplane, worker)
│   │   └── components/           #   reusable feature/provider Talos fragments selected by Ansible
│   ├── clusters/
│   │   └── kube1/                # USER-OWNED — kube1 selected-cluster active source of truth (ADR-016)
│   │       ├── cluster.yaml      #   provider, features, versions, topology, Flux repo
│   │       ├── nodes.yaml        #   per-node hardware identities
│   │       └── talos/            #   kube1-specific Talos overlays (all, controlplane, worker, per-node)
│   └── overrides/                # LEGACY FALLBACK — migration compatibility only (see ADR-016)
│       ├── cluster.yaml
│       └── nodes.yaml
├── infra/                        # VM provisioning and cluster bootstrap
│   ├── ansible/
│   │   ├── inventories/
│   │   │   ├── common/           # Provider-agnostic — loaded on every run
│   │   │   │   └── group_vars/all/      # GENERATED by render-config.yml — do not edit
│   │   │   │       ├── cluster.yml      #   cluster_name, VIP, endpoint, kubernetes_version, node_role_short map
│   │   │   │       ├── talos.yml        #   talos_version, talos_extensions (schematic resolved at runtime)
│   │   │   │       └── flux.yml         #   Flux repo URL/branch/path
│   │   │   ├── providers/
│   │   │   │   ├── hcloud/       # Hetzner Cloud provider — load with -i inventories/providers/hcloud
│   │   │   │   │   ├── hcloud.yml    # Dynamic inventory plugin: label-filtered, compose node_*_ip
│   │   │   │   │   └── group_vars/all/
│   │   │   │   │       ├── hcloud.yml       # GENERATED — location, server types, topology, firewall, ...
│   │   │   │   │       └── hcloud.sops.yaml # USER/SECRET-OWNED — SOPS-encrypted hcloud_token (never rendered)
│   │   │   │   └── manual/       # Static inventory — load with -i inventories/providers/manual
│   │   │   │       ├── hosts.yml            # Working 3-node hybrid example; fill in real IPs
│   │   │   │       └── group_vars/all/      # GENERATED — provider-keyed bootstrap_pre_flux_components etc.
│   │   ├── playbooks/
│   │   │   ├── render-config.yml     # Local: render config/* into generated group_vars + Talos patches + Flux base
│   │   │   ├── bootstrap-talos.yml   # Provider-agnostic: gen configs, apply, etcd, converge image
│   │   │   ├── upgrade-talos.yml     # Serial per-node OS upgrade with health checks
│   │   │   ├── bootstrap-flux.yml    # Pre-Flux add-ons (Cilium CNI, hcloud CCM) then flux bootstrap git
│   │   │   └── providers/
│   │   │       └── hcloud/               # Hetzner-only: require -i inventories/providers/hcloud
│   │   │           ├── image-talos.yml   # Build Talos disk snapshot (run once per version/schematic)
│   │   │           └── provision-infra.yml  # Create VMs, private network, firewall
│   │   └── roles/
│   │       ├── config_render/    # The compiler: load → validate → compute → render (Ansible/Talos/Flux) → check
│   │       ├── talos_schematic/  # Resolve extensions → schematic ID via factory.talos.dev; cache result
│   │       ├── talos_config/     # Apply Talos machine configs (composes the generated layered patch set)
│   │       ├── talos_upgrade/    # Serial per-node image check + talosctl upgrade
│   │       ├── providers/
│   │       │   └── hcloud/
│   │       │       └── hcloud_servers/   # hcloud API: VMs, firewall, private network (topology-driven)
│   │       └── flux_bootstrap/   # Pre-Flux add-ons + flux bootstrap git: preflight, namespaces, cilium, hcloud_ccm, flux
│   └── talos/
│       ├── patches/
│       │   └── generated/        # GENERATED — computed integration patches only (human-authored Talos YAML from config/talos/ + config/clusters/<cluster>/talos/)
│       │       ├── all/              # applied to every node
│       │       ├── controlplane/     # applied to control-plane (etcd members) only
│       │       ├── worker/           # applied to worker-only nodes
│       │       └── nodes/            # per-node machine-level overrides (named by hostname)
│       ├── secrets/              # Rendered machine configs + talosctl secrets — gitignored
│       ├── schematic_id          # Resolved schematic ID cache — gitignored
│
├── flux/                         # FluxCD GitOps — all cluster resources
│   ├── clusters/
│   │   └── kube1/                # Flux entrypoint for this cluster
│   │       ├── infrastructure.yaml
│   │       └── apps.yaml
│   ├── infrastructure/
│   │   ├── controllers/          # Catalog / render-owned base / user overlay (ADR-010, ADR-012)
│   │   │   ├── _components/         # TEMPLATE-OWNED catalog (cilium + providers/hcloud/ccm on kube1; future csi, cert-manager, ingress, openebs)
│   │   │   ├── base/                # GENERATED — enabled components for this cluster (cilium + hcloud-ccm on kube1)
│   │   │   ├── patches/             # USER-OWNED — kustomize strategic-merge patches
│   │   │   └── kustomization.yaml   # USER-OWNED overlay (references base/ + patches/)
│   │   └── configs/              # ClusterIssuers, StorageClasses, etc.
│   └── apps/                     # Workload Helm releases and manifests
│
└── docs/                         # Architecture decisions and runbooks
    ├── node-roles.md             # Role model, naming convention, Kubernetes labels
    ├── upgrades.md               # Talos version and schematic upgrade procedure
    └── adding-a-provider.md      # Inventory contract for bringing up nodes from a new provider
```

## Implementation status

The Architecture table records *decisions*; not all of them are built yet. As of now:

- **Working — provisioning & Talos:** Talos image build (`providers/hcloud/image-talos.yml`, schematic-from-config via `talos_schematic` role), VM / network / firewall provisioning (`providers/hcloud/provision-infra.yml`, topology-driven from `hcloud_topology`), Talos + etcd bootstrap and kubeconfig fetch (`bootstrap-talos.yml`), serial per-node Talos OS upgrades (`upgrade-talos.yml`). Both `providers/hcloud` and `providers/manual` inventory paths are wired up (ADR-014).
- **Working — config compiler (ADR-010 / 011 / 012 / 016):** `playbooks/render-config.yml` + the `config_render` role are built. `config/defaults/` (template-owned) is deep-merged with the selected cluster config at `config/clusters/<cluster>/` (user-owned, selected via `-e config_cluster=<name>`) — overrides win per-key — then validated and rendered into the existing `inventories/.../group_vars/`, the layered Talos patch set under `infra/talos/patches/generated/{all,controlplane,worker,nodes}/`, and the render-owned `flux/infrastructure/controllers/generated/selected/kustomization.yaml` (cilium + hcloud-ccm on kube1). Talos-shaped inputs are composed from `config/talos/base/` + `config/talos/components/` (provider/feature-selected) + `config/clusters/<cluster>/talos/` (cluster-specific overlays). `config/overrides/` remains as a legacy migration fallback only. Render check mode (`-e config_render_check=true` or `--tags check`) renders to a temp dir and diffs against the committed generated tree. Generated files start with `# Generated by render-config; do not edit.` — edit the selected-cluster config and Talos overlays, not the generated outputs.
- **Working — pre-Flux + Flux (Cilium, hcloud CCM, Flux adoption):** `playbooks/bootstrap-flux.yml` runs the `flux_bootstrap` role, which is now a real implementation (preflight → namespaces → cilium → hcloud_ccm → flux) — not a stub. The role installs the **Cilium** Helm release into the `cilium-system` namespace and the **hcloud Cloud Controller Manager** Helm release into the `hcloud-ccm-system` namespace, then runs `flux bootstrap git` to install Flux controllers in `flux-system` and adopt the same releases from the render-generated `generated/selected/kustomization.yaml`. The render-owned `generated/selected/kustomization.yaml` references `_components/cilium` and `_components/providers/hcloud/ccm`; both catalog entries have matching `Namespace` + `HelmRepository` + `HelmRelease` (chart versions pinned). The Talos all-node patch `infra/talos/patches/generated/all/10-external-cloud-provider.yaml.j2` is generated and applied to set `cluster.externalCloudProvider.enabled: true` (so the CCM can set `spec.providerID` and clear the uninitialized taint). The pre-Flux component list (`bootstrap_pre_flux_components`) is computed by the compiler from `cni.name` + `cloud_controller_manager.enabled` + `provider.name`; the hcloud provider renders `[cilium, hcloud-ccm]`, the manual provider renders `[cilium]` (no CCM, no external cloud provider patch).
- **Still deferred:** Hetzner CSI is *not* installed — `features.hcloudCsi: false` and no catalog entry. cert-manager, OpenEBS, and ingress are *not* installed — their `features.*` flags default to `false`/`none` and no catalog entries exist. `flux/infrastructure/configs/` and `flux/apps/` are still empty `resources: []` placeholders. The `talosctl` deviceSelector probe (ADR-013) for replacing the current `interface: eth1` form is also deferred. **Live cluster state is unverified** — every item in this "Working" section has been reviewed, but the user has not yet run `bootstrap-flux.yml` against a live cluster, so "working" means "the code is built and passes review/validation" not "we have observed it running".
- **One missing user input:** `flux.repoUrl` is still empty in `config/clusters/kube1/cluster.yaml` — `flux bootstrap git` will not run until the user fills it in (and the matching authentication method is configured — SSH deploy key or HTTPS credentials). The other pre-Flux components do not block on it: `bootstrap-flux.yml` is a single Ansible run, but the `flux` step is gated on `flux_repo_url` being non-empty.

Keep this section honest — update it as pieces land, and never describe a planned component as if it's already running.

## Prerequisites

```bash
brew install hcloud talosctl kubectl ansible ansible-lint helm flux age sops

# Install the pinned collections (hetzner.hcloud, community.sops) into
# infra/ansible/.collections/ — project-local per ansible.cfg's collections_path.
# Re-run this after any requirements.yml bump.
ansible-galaxy collection install -r infra/ansible/requirements.yml

# Generate your personal age keypair (one-time, machine-local). SOPS reads this
# default macOS path automatically — no SOPS_AGE_KEY_FILE needed.
age-keygen -o "$HOME/Library/Application Support/sops/age/keys.txt"
# Send the printed "public key: age1..." to whoever maintains .sops.yaml so your
# key is added as a recipient — without that, `sops -d` on this repo's secrets fails.

# Authenticate the hcloud CLI (for manual `hcloud` commands)
hcloud context create kube1    # paste Hetzner API token when prompted
hcloud server list             # verify
```

Ansible itself does **not** read the hcloud CLI context — it takes the same token from
the SOPS-encrypted `group_vars/hcloud.sops.yaml` as `hcloud_token`. See
[Secrets policy](#secrets-policy).

## Key workflows

All Ansible commands are run from `infra/ansible/`. Select provider with double `-i`:
`-i inventories/common -i inventories/providers/hcloud` (Hetzner) or `-i inventories/common -i inventories/providers/manual` (bare metal / other cloud).

### Full cluster bootstrap from scratch (Hetzner)
```bash
cd infra/ansible

# 0. Build the Talos snapshot — once per Talos version + extensions combination.
#    Boots a temporary builder server, dd's the factory image, snapshots, deletes
#    the server. Builder type is the first available from hcloud_builder_server_types.
#    Snapshot is named talos-v1.13.3-ce4c980550dd (version + schematic short-hash)
#    and labelled with talos-version + talos-schematic-id for content-addressed lookup.
#    provision-infra.yml resolves the snapshot ID automatically from these labels —
#    no manual ID copy needed.
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/image-talos.yml

# 1. Provision VMs and the hcloud private network
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/provision-infra.yml

# 2. Bootstrap Talos and etcd, fetch the kubeconfig
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml

# 3. Install pre-Flux add-ons (Cilium CNI, hcloud CCM) and bootstrap Flux
#    from the configured git repo. Requires flux.repoUrl to be set in
#    config/clusters/kube1/cluster.yaml (see Implementation status).
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml
```

After this playbook finishes, the user should validate that:

```bash
# Cilium is Ready in cilium-system, with kube-proxy replacement enabled
kubectl -n cilium-system rollout status daemonset/cilium
kubectl -n cilium-system get pods -o wide

# hcloud CCM cleared the uninitialized taint and set providerID on every node.
# Expected: no line contains "node.cloudprovider.kubernetes.io/uninitialized"
# in the taints output, and every providerID starts with "hcloud://".
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" providerID="}{.spec.providerID}{" taints="}{.spec.taints}{"\n"}{end}'

# Flux controllers in flux-system are Ready, and the kustomizations
# infra-controllers / infra-configs / apps are all Ready.
kubectl -n flux-system get pods
flux get kustomizations -A
flux get helmreleases -A
```

If any check fails, do **not** proceed to install workloads — the next pre-Flux
component (CCM in particular) is a hard prerequisite for every other pod
scheduling. See [Pre-Flux bootstrap](#pre-flux-bootstrap) for the taint and
namespace contract behind these commands.

### Bootstrap from a manual / other-cloud inventory
```bash
cd infra/ansible
# Edit inventories/providers/manual/hosts.yml — fill in real IPs and node_role per node.
# bootstrap-talos.yml converges the Talos OS image on first run (manual nodes may
# boot a generic ISO; the role upgrades them to the factory schematic image).
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml

# Manual resolves ccm to disabled — bootstrap-flux.yml installs only Cilium,
# then bootstraps Flux. Same flux.repoUrl precondition as the hcloud flow.
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-flux.yml
```

### Upgrade Talos version or change extensions
```bash
# 1. Edit talos.version / talos.extensions in config/clusters/kube1/cluster.yaml,
#    then re-render: ansible-playbook playbooks/render-config.yml -e config_cluster=kube1
# 2. (hcloud) build a new snapshot — image-talos.yml checks labels first and skips
#    the build if a matching snapshot already exists; delete the old snapshot from
#    Hetzner Cloud console if you want to force a rebuild.
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/image-talos.yml
# 3. Run the upgrade playbook — sequences one node at a time, health-checks between
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/upgrade-talos.yml
```

See `docs/upgrades.md` for the detailed procedure including rollback guidance.

### Push a config-only change (no OS upgrade)

The render compiler is the source of truth: edit the selected-cluster config and Talos overlays, render, then push.

```bash
# 1. Edit the user-owned input — config/clusters/kube1/cluster.yaml (or
#    config/clusters/kube1/talos/*.yaml for Talos machine config overlays,
#    ADR-016). NEVER edit files marked
#    "# Generated by render-config; do not edit." — the next render overwrites them.
cd infra/ansible

# 2. Re-render generated group_vars, Talos patches, and the Flux base.
ansible-playbook playbooks/render-config.yml -e config_cluster=kube1
# Review the generated diff in git before continuing. Use
# `-e config_render_check=true` to render to a temp dir and diff without writing.

# 3. Push to running nodes:
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml
# Re-applies configs with --mode=no-reboot. If a change requires a reboot
# (kernel param etc.), Talos reports it; run upgrade-talos.yml to roll it.
```

### Recovering from an IP-based lock-out

If your home/admin IP changes and `hcloud_firewall_open_access: false` is set, the kube1 firewall silently stops admitting talosctl/kubectl from your new IP — Talos has no SSH fallback. To recover:

1. **Set `hcloud.firewallOpenAccess: true`** in `config/clusters/kube1/cluster.yaml` and re-render:
   ```bash
   cd infra/ansible
   ansible-playbook playbooks/render-config.yml -e config_cluster=kube1
   ```
   This opens ports 50000 and 6443 to the entire internet (0.0.0.0/0 + ::/0). The APIs remain mTLS/cert-authenticated, so "open" only widens network reach, not auth.
   *Emergency shortcut:* edit the generated `inventories/providers/hcloud/group_vars/all/hcloud.yml` directly (set `hcloud_firewall_open_access: true`) and skip the render step — but re-render from the cluster config later to avoid drift.

2. **Re-run provision-infra.yml** to apply the new firewall rules:
   ```bash
   cd infra/ansible
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/provision-infra.yml
   ```

3. **Once access is restored**, edit `config/clusters/kube1/cluster.yaml`, then either:
   - Set `hcloud.adminIps: ["<your-new-ip>/32"]` and `hcloud.firewallOpenAccess: false` for IP-scoped access (recommended if you have a stable IP), then re-render, or
   - Leave `hcloud.firewallOpenAccess: true` if your IP is dynamic (accepts the scanning risk)

The Talos/Kubernetes APIs are still mTLS/cert-authenticated when "open" — opening them only widens *network* reach, not auth — but it does expose the endpoint to internet-wide scanning, so prefer IP-scoped access when your IP is stable.

### Force Flux to reconcile immediately
```bash
flux reconcile kustomization flux-system --with-source
```

### Run a single Ansible role
```bash
cd infra/ansible
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/provision-infra.yml --tags hcloud_servers
```

## Engineering standards

Match the conventions already in the repo — they're deliberate, not accidental.

### Ansible
- **FQCN everywhere** — `ansible.builtin.*`, `hetzner.hcloud.*`; no short module names.
- **Idempotency is mandatory.** Use `creates:`, `changed_when`, `failed_when`, and `until`/`retries`/`delay` so a second run is a clean no-op and a task only reports `changed` when it actually changed something. Use `check_mode: false` only for read-only tasks that can't run under `--check`, and comment why.
- **Variable namespacing.** Role-local vars are prefixed with the role name (`hcloud_servers_*`, `talos_config_*`). Cross-cutting `group_vars` use domain prefixes (`hcloud_`, `talos_`, `cluster_`, `flux_`, `kubernetes_`).
- **Secrets via `module_defaults`** (`group/hetzner.hcloud.all` → `api_token`) — never inline in a task, never committed.
- **Comment the *why*,** especially for workarounds and API quirks — match the existing comment density (the `# noqa` reasons, the maintenance-mode detection, the etcd member-IP note set the bar).
- **Pin versions:** collections in `requirements.yml`, tool/image versions in `group_vars`.

### Talos & Flux
- The machine config is layered (ADR-016) from Talos-shaped patch manifests under `config/talos/` and `config/clusters/<cluster>/talos/`, with computed integration fragments rendered into `infra/talos/patches/generated/`. The renderer reads the selected-cluster config at `config/clusters/<cluster>/cluster.yaml` and its `talos/` overlays — never hand-edit the generated patch files; the next render overwrites them. Final machine configs are rendered into the gitignored `secrets/` dir by `talosctl gen config` + the patch set; never commit them — they embed the cluster CA and keys.
- Every Flux `Kustomization` pins `interval`, sets `prune: true`, and declares explicit `dependsOn` ordering (controllers → configs → apps); infra layers set `wait: true` so dependents block until they're Ready. The `flux/infrastructure/controllers/kustomization.yaml` is a user-owned overlay; its `base/kustomization.yaml` is render-generated (ADR-011, ADR-012).
- Pin Helm chart and image versions — no `:latest`, no floating tags.

## Talos specifics

- Machine configs are generated via `talosctl gen config` and then composed with the layered patch set in `infra/talos/patches/generated/` (ADR-016). Never commit the rendered machine configs that contain secrets.
- The patch stack is composed from `config/talos/base/`, `config/clusters/<cluster>/talos/`, and `config/talos/components/` selected by provider/features (ADR-016). Ansible composes the ordered patch list and passes it to `talosctl gen config`. Edit the selected-cluster config and Talos overlays, render with `-e config_cluster=<name>`, then push with `bootstrap-talos.yml` — hand-editing the generated patch files is a no-op, the next render overwrites them.
- The VIP is set in the `network.interfaces` section of the machine config patch. All 3 nodes declare the same VIP; Talos elects the holder via ARP gratuitous announcements.
- No SSH on Talos nodes. Use `talosctl dmesg`, `talosctl logs`, and `talosctl console` for debugging.
- Installer images come from the Talos Image Factory (`factory.talos.dev`), not from `ghcr.io/siderolabs`. The schematic encodes the extension set; `talos_schematic` role resolves it to a content-addressed ID at runtime and caches it in `infra/talos/schematic_id`. Never hardcode the schematic ID — edit `talos.extensions.base` / `talos.extensions.extra` in `config/clusters/<cluster>/cluster.yaml` and re-render instead.
- `allowSchedulingOnControlPlanes: true` is applied only to hybrid nodes (via per-node inline `--config-patch` at apply-config time). Pure controlplane nodes keep the default NoSchedule taint.

## Flux specifics

- Cilium must be installed before Flux reconciliation starts — it is a CNI dependency that prevents pods from scheduling. The `bootstrap-flux.yml` playbook runs `preflight → namespaces → cilium → hcloud_ccm → flux`; the `flux` step is the last one and only runs once the pre-Flux components are Ready. See [Pre-Flux bootstrap](#pre-flux-bootstrap) below for the namespace and taint contract behind that order.
- The `flux/clusters/kube1/` directory is the Flux entrypoint; it references `infrastructure` and `apps` kustomizations with explicit dependency ordering (`dependsOn`).
- `infrastructure/controllers` deploys before `infrastructure/configs` (configs depend on CRDs from controllers).
- `apps` depends on `infrastructure` being ready.
- `flux/infrastructure/controllers/` follows the catalog / render-owned base / user overlay split from ADR-010/012: `_components/` is the template-owned catalog (cilium + providers/hcloud/ccm on kube1 today; future csi, cert-manager, ingress, openebs), `base/kustomization.yaml` is render-generated from the selected-cluster config (ADR-016), and `patches/` + the top-level `kustomization.yaml` are user-owned.
- Pre-Flux components and Flux-managed HelmReleases use the **same** release name, namespace, chart, and values. Flux adopts an existing Helm release on first reconciliation as long as `install.disableTakeOwnership` is left at its default `false` — a mismatch (e.g. renaming `cilium` to `cilium-cn`) makes Helm uninstall and reinstall, which can drop in-flight pod networking. The `flux_bootstrap` role's `defaults/main.yml` and `_components/<name>/helmrelease.yaml` must stay in lock-step — never edit one without the other.

## Pre-Flux bootstrap

`playbooks/bootstrap-flux.yml` runs the `flux_bootstrap` role, which performs five steps in order: **preflight → namespaces → cilium → hcloud_ccm → flux**. Each step is gated on the pre-Flux component list, which the config compiler computes from `cni.name` + `cloud_controller_manager.enabled` + `provider.name`. The hcloud provider resolves to `[cilium, hcloud-ccm]`; manual resolves to `[cilium]` (no CCM, no `externalCloudProvider` patch).

### Why this order

- **Cilium before Flux.** Flux controllers are pods. Without a CNI, no pod can schedule and Flux cannot reconcile. Cilium goes first so by the time `flux bootstrap git` installs the controllers, pod networking is already up. Cilium is reached via Talos KubePrism at `localhost:7445` — a host-network sidecar that is always reachable, avoiding the chicken-and-egg problem where Cilium cannot reach the API via `ClusterIP` because kube-proxy is disabled.
- **CCM after Cilium, before Flux.** The hcloud CCM needs pod networking to talk to the K8s API, and it has a hard contract with Talos's external-cloud-provider mode that the `flux_bootstrap` role explicitly checks (see below). If CCM install fails, new workload scheduling is blocked — CCM is a hard prerequisite for every other pod. The hcloud CCM's `networking.enabled: false` (ADR-005) — no route controller, no LoadBalancer controller — because Cilium owns all routing (VXLAN tunnel) and ingress is host-network Envoy Gateway (ADR-004).

### Namespaces

Each pre-Flux component runs in its own dedicated `<component>-system` namespace, not in `kube-system`. Ansible creates the namespace and Flux takes ownership through its `Namespace` resource in the catalog:

| Component | Namespace | Owner pre-Flux | Owner post-Flux |
|---|---|---|---|
| Cilium | `cilium-system` | Ansible (Helm install) | Flux HelmRelease adoption |
| hcloud CCM | `hcloud-ccm-system` | Ansible (Secret + Helm install) | Flux HelmRelease adoption |
| Flux controllers | `flux-system` | `flux bootstrap git` | `flux bootstrap git` |

### External cloud provider taint (hcloud only)

When `cluster.externalCloudProvider.enabled: true` is applied via the generated Talos patch (`infra/talos/patches/generated/all/10-external-cloud-provider.yaml.j2`), Talos runs kubelet and kube-controller-manager with `--cloud-provider=external` and Kubernetes taints every node with `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule`. Only a CCM can clear this taint. Until it does, no workloads schedule — including Flux's own controllers — so the CCM *must* run before `flux bootstrap`.

The `flux_bootstrap` role enforces this contract with two readiness checks after the CCM deployment reports Ready (it retries for ~2 minutes total):

```bash
# 1. No node may carry the uninitialized taint.
kubectl --kubeconfig infra/talos/secrets/kubeconfig get nodes \
  -o go-template='{{ range .items }}{{ range .spec.taints }}{{ .key }}{{ "\n" }}{{ end }}{{ end }}'
# Expected: no "node.cloudprovider.kubernetes.io/uninitialized" line.

# 2. Every node must have spec.providerID starting with "hcloud://".
kubectl --kubeconfig infra/talos/secrets/kubeconfig get nodes \
  -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}'
# Expected: every line starts with "hcloud://".
```

A separate Cilium-specific caveat: the Cilium **operator** deployment does not tolerate the uninitialized taint by default (Cilium issue #41921). Without an explicit toleration, the operator can't schedule while CCM is still clearing the taint, and the CNI/CCM bootstrap chain deadlocks. The Flux HelmRelease (and the Ansible `flux_bootstrap_cilium_values`) re-states the operator's full toleration list including the uninitialized key — `operator.tolerations` in Helm values replaces rather than merges, so the chart defaults must be re-declared.

### hcloud token Secret ownership

The hcloud API token never lands in Git. The flow is:

1. SOPS-encrypted `infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml` holds the token as `hcloud_token`.
2. Ansible's `flux_bootstrap` role creates Kubernetes Secret `hcloud-ccm-system/hcloud` (key `token`) from `hcloud_token` with `no_log: true`. The `kubernetes.core.k8s` module is idempotent — re-runs update the Secret in place rather than failing.
3. The hcloud CCM chart reads its token from that Secret via its default `envFrom` — the Flux HelmRelease does not own a Secret manifest, so no plaintext token ever reaches Git.

### Live validation after `bootstrap-flux.yml`

The user runs these in their own terminal — agents must not run live cluster operations:

```bash
# Cilium Ready in cilium-system
kubectl -n cilium-system rollout status daemonset/cilium
kubectl -n cilium-system get pods -o wide

# hcloud CCM cleared the uninitialized taint and set providerID
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" providerID="}{.spec.providerID}{" taints="}{.spec.taints}{"\n"}{end}'
# Expected: no "node.cloudprovider.kubernetes.io/uninitialized" in any taints line;
# every providerID starts with "hcloud://".

# Flux controllers Ready, infra-controllers kustomization Ready
kubectl -n flux-system get pods
flux get kustomizations -A
flux get helmreleases -A
# Expected: cilium and hcloud-cloud-controller-manager are listed as Ready
# HelmReleases with no install/uninstall churn (adoption was clean).

# Helm CLI cross-check: the same releases exist with the same names
helm -n cilium-system list
helm -n hcloud-ccm-system list
```

If any check fails, do **not** proceed to install workloads — the next pre-Flux component (CCM in particular) is a hard prerequisite for every other pod scheduling.

## Secrets policy

Never commit:
- `infra/talos/secrets/` — talosconfig and machine secrets generated by `talosctl gen config`
- `infra/talos/schematic_id` — resolved schematic ID cache (gitignored)

The hcloud API token lives encrypted at
`infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml`. Committed via
[SOPS](https://github.com/getsops/sops) + [age](https://github.com/FiloSottile/age).
`ansible.cfg`'s `vars_plugins_enabled = host_group_vars,community.sops.sops` makes
Ansible decrypt it automatically at run time — no extra steps for `ansible-playbook`.
A future second provider would get its own `inventories/providers/<provider>/group_vars/all/<provider>.sops.yaml`.

To edit the secret:
```bash
sops infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml
```
This opens `$EDITOR` with the *decrypted* content; SOPS re-encrypts on save. Your age
private key must be at `~/Library/Application Support/sops/age/keys.txt` (see
Prerequisites) and your age public key must be listed as a recipient in `.sops.yaml`.

Recipients are managed in `/.sops.yaml` (repo root) — add a new `age1...` public key
there (and re-run `sops updatekeys infra/ansible/inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml`)
to grant another machine/person decrypt access.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- Use graphify automatically for repo discovery before spawning broad file-search work. If the graph answer is sufficient, do not run raw source searches.
- Do not use the generic `explore` agent for kube1 repo questions. Use graphify first, then LSP/targeted reads only after graphify narrows the area.
- The top-level session should not run raw shell repo searches such as `rg`, `grep`, `find`, or `echo ... && rg ... | head`.
- Use `web-fast-context` for official docs and external facts.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
