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
| `repo-fast-context` | Read-only, low-cost repo context fetcher. Use after graphify only when additional file/line context is needed. Prefer `rg`/ripgrep for raw fallback searches. | haiku / low |
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
- **Use graphify before raw repo search.** For codebase questions, query graphify first. Use `repo-fast-context` only when graphify is insufficient, stale for the area, absent, or file/line evidence is needed. When both repo and external-doc context are needed, run `repo-fast-context` and `web-fast-context` in parallel.
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
| Hetzner CCM | Not deployed — no `LoadBalancer`-type services needed yet |
| Ingress | TBD |

**Why single DC:** Talos native VIP is ARP-based (L2). Hetzner private networks are L3-routed — ARP does not propagate across DC boundaries even within `eu-central`. Multi-DC would require a Hetzner LB for the API endpoint.

## Repository structure

```
kube1/
├── infra/                        # VM provisioning and cluster bootstrap
│   ├── ansible/
│   │   ├── inventories/
│   │   │   ├── common/           # Provider-agnostic — loaded on every run
│   │   │   │   └── group_vars/all/
│   │   │   │       ├── cluster.yml  # cluster_name, VIP, endpoint, kubernetes_version, node_role_short map
│   │   │   │       ├── talos.yml    # talos_version, talos_extensions list (schematic resolved at runtime)
│   │   │   │       └── flux.yml     # Flux repo URL/branch/path
│   │   │   ├── providers/
│   │   │   │   ├── hcloud/       # Hetzner Cloud provider — load with -i inventories/providers/hcloud
│   │   │   │   │   ├── hcloud.yml    # Dynamic inventory plugin: label-filtered, compose node_*_ip
│   │   │   │   │   └── group_vars/all/
│   │   │   │   │       ├── hcloud.yml       # location, server type, topology counts, firewall, ...
│   │   │   │   │       └── hcloud.sops.yaml # SOPS-encrypted hcloud_token (committed)
│   │   │   │   └── manual/       # Static inventory — load with -i inventories/providers/manual
│   │   │   │       └── hosts.yml # Working 3-node hybrid example; fill in real IPs to use
│   │   ├── playbooks/
│   │   │   ├── bootstrap-talos.yml   # Provider-agnostic: gen configs, apply, etcd, converge image
│   │   │   ├── upgrade-talos.yml     # Serial per-node OS upgrade with health checks
│   │   │   ├── bootstrap-flux.yml    # Cilium pre-Flux + Flux bootstrap (STUB — not built yet)
│   │   │   └── providers/
│   │   │       └── hcloud/               # Hetzner-only: require -i inventories/providers/hcloud
│   │   │           ├── image-talos.yml   # Build Talos disk snapshot (run once per version/schematic)
│   │   │           └── provision-infra.yml  # Create VMs, private network, firewall
│   │   └── roles/
│   │       ├── talos_schematic/  # Resolve extensions → schematic ID via factory.talos.dev; cache result
│   │       ├── talos_config/     # Generate + apply Talos machine configs (per-role, per-node patches)
│   │       ├── talos_upgrade/    # Serial per-node image check + talosctl upgrade
│   │       ├── providers/
│   │       │   └── hcloud/
│   │       │       └── hcloud_servers/   # hcloud API: VMs, firewall, private network (topology-driven)
│   │       └── flux_bootstrap/   # Flux bootstrap into the cluster (stub)
│   └── talos/
│       ├── patches/              # Jinja templates: common.yaml.j2, controlplane.yaml.j2
│       ├── secrets/              # Rendered configs + talosctl secrets — gitignored
│       ├── schematic_id          # Resolved schematic ID cache — gitignored
│
├── flux/                         # FluxCD GitOps — all cluster resources
│   ├── clusters/
│   │   └── kube1/                # Flux entrypoint for this cluster
│   │       ├── infrastructure.yaml
│   │       └── apps.yaml
│   ├── infrastructure/
│   │   ├── controllers/          # Cilium, cert-manager, CSI driver, OpenEBS
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

- **Working:** Talos image build (`providers/hcloud/image-talos.yml`, schematic-from-config via `talos_schematic` role), VM / network / firewall provisioning (`providers/hcloud/provision-infra.yml`, topology-driven from `hcloud_topology`), Talos + etcd bootstrap and kubeconfig fetch (`bootstrap-talos.yml`), serial per-node Talos OS upgrades (`upgrade-talos.yml`). Both `providers/hcloud` and `providers/manual` inventory paths are wired up.
- **Not built yet:** Flux is *not* bootstrapped — the `flux_bootstrap` role is an empty stub and `flux_repo_url` is unset in `inventories/common/group_vars/all/flux.yml`. Cilium is *not* installed (the "Cilium pre-Flux" step in `bootstrap-flux.yml` is planned, not implemented). The `flux/infrastructure/{controllers,configs}` and `flux/apps` kustomizations are empty `resources: []` placeholders.

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

# 3. Install Cilium (pre-Flux CNI) + bootstrap Flux — NOT BUILT YET
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml
```

### Bootstrap from a manual / other-cloud inventory
```bash
cd infra/ansible
# Edit inventories/providers/manual/hosts.yml — fill in real IPs and node_role per node.
# bootstrap-talos.yml converges the Talos OS image on first run (manual nodes may
# boot a generic ISO; the role upgrades them to the factory schematic image).
ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml
```

### Upgrade Talos version or change extensions
```bash
# 1. Edit talos_version / talos_extensions in inventories/common/group_vars/all/talos.yml
# 2. (hcloud) build a new snapshot — image-talos.yml checks labels first and skips
#    the build if a matching snapshot already exists; delete the old snapshot from
#    Hetzner Cloud console if you want to force a rebuild.
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/image-talos.yml
# 3. Run the upgrade playbook — sequences one node at a time, health-checks between
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/upgrade-talos.yml
```

See `docs/upgrades.md` for the detailed procedure including rollback guidance.

### Push a config-only change (no OS upgrade)
```bash
# Edit infra/talos/patches/common.yaml.j2 or controlplane.yaml.j2, then:
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml
# Re-applies configs to running nodes with --mode=no-reboot. If a change requires
# a reboot (kernel param etc.), Talos reports it; run upgrade-talos.yml to roll it.
```

### Recovering from an IP-based lock-out

If your home/admin IP changes and `hcloud_firewall_open_access: false` is set, the kube1 firewall silently stops admitting talosctl/kubectl from your new IP — Talos has no SSH fallback. To recover:

1. **Set `hcloud_firewall_open_access: true`** in `inventories/providers/hcloud/group_vars/all/hcloud.yml` — this opens ports 50000 and 6443 to the entire internet (0.0.0.0/0 + ::/0). The APIs remain mTLS/cert-authenticated, so "open" only widens network reach, not auth.

2. **Re-run provision-infra.yml** to apply the new firewall rules:
   ```bash
   cd infra/ansible
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/provision-infra.yml
   ```

3. **Once access is restored**, either:
   - Set `hcloud_admin_ips: ["<your-new-ip>/32"]` and `hcloud_firewall_open_access: false` for IP-scoped access (recommended if you have a stable IP), or
   - Leave `hcloud_firewall_open_access: true` if your IP is dynamic (accepts the scanning risk)

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
- The machine config is built from the Jinja patch template, rendered into the gitignored `secrets/` dir. Never commit a rendered config — it embeds the cluster CA and keys.
- Every Flux `Kustomization` pins `interval`, sets `prune: true`, and declares explicit `dependsOn` ordering (controllers → configs → apps); infra layers set `wait: true` so dependents block until they're Ready.
- Pin Helm chart and image versions — no `:latest`, no floating tags.

## Talos specifics

- Machine configs are generated via `talosctl gen config` and stored as patches — never commit generated configs that contain secrets.
- The VIP is set in the `network.interfaces` section of the machine config patch. All 3 nodes declare the same VIP; Talos elects the holder via ARP gratuitous announcements.
- No SSH on Talos nodes. Use `talosctl dmesg`, `talosctl logs`, and `talosctl console` for debugging.
- Installer images come from the Talos Image Factory (`factory.talos.dev`), not from `ghcr.io/siderolabs`. The schematic encodes the extension set; `talos_schematic` role resolves it to a content-addressed ID at runtime and caches it in `infra/talos/schematic_id`. Never hardcode the schematic ID — edit `talos_extensions` in `inventories/common/group_vars/all/talos.yml` instead.
- `allowSchedulingOnControlPlanes: true` is applied only to hybrid nodes (via per-node inline `--config-patch` at apply-config time). Pure controlplane nodes keep the default NoSchedule taint.

## Flux specifics

- Cilium must be installed before Flux reconciliation starts — it is a CNI dependency that prevents pods from scheduling.
- The `flux/clusters/kube1/` directory is the Flux entrypoint; it references `infrastructure` and `apps` kustomizations with explicit dependency ordering (`dependsOn`).
- `infrastructure/controllers` deploys before `infrastructure/configs` (configs depend on CRDs from controllers).
- `apps` depends on `infrastructure` being ready.

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
- Do not use the generic `explore` agent for kube1 repo questions. Use graphify first, then `repo-fast-context` only when extra context is required.
- The top-level session should not run raw shell repo searches such as `rg`, `grep`, `find`, or `echo ... && rg ... | head`. Delegate that work to `repo-fast-context` after graphify.
- Use `repo-fast-context` only when graphify does not surface enough detail, when the graph is absent/stale for the asked area, or when specific file/line evidence is needed.
- `repo-fast-context` should prefer `rg`/ripgrep when available for raw fallback searches. Use targeted patterns and return concise file/line references, not large dumps.
- For independent context needs, run `repo-fast-context` and `web-fast-context` in parallel so repo discovery and official-doc lookup do not block each other.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
