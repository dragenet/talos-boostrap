# Provider-portability refactor plan

Date: 2026-06-22  
Slug: `provider-portability-refactor`  
Planning context constraint: this plan uses only `docs/adding-a-provider.md`, `docs/node-roles.md`, and `.ai/review-2026-06-10.md` as requested. No implementation files were inspected or changed.

## Goal and scope

Implement the refactor implied by review/ADR notes A-D:

1. Make the provider seam explicit: provider-specific provisioning produces a small node contract; provider-agnostic Talos/Flux playbooks consume it.
2. Rename/reorganize playbooks so provider-bound playbooks are clearly separated from provider-neutral bootstrap playbooks.
3. Split mixed `group_vars` by layer/domain while preserving existing variable prefixes.
4. Replace `node_ips.yml` filesystem IPC with inventory-owned node IP discovery, using Hetzner dynamic inventory for hcloud and static hostvars for manual inventories.

Out of scope:

- No Flux feature implementation beyond preserving/renaming the bootstrap entrypoint.
- No live infrastructure provisioning, Talos bootstrap, upgrades, or kubectl/flux reconciliation as part of implementation without explicit user approval.
- No secrets rotation or broader review findings C1/C2/H1/H2/M1-M5 unless they directly block this refactor.

## Current-state facts from the allowed repo context

- `docs/adding-a-provider.md` defines the provider-neutral inventory contract: every node must expose `node_role`, `node_public_ip`, and `node_private_ip`; inventory hostname is the Talos/Kubernetes node name.
- The same doc states cluster commands are local (`ansible_connection: local`) and should use `ansible_playbook_python`, with those common settings loaded from `inventories/common/group_vars/all/cluster.yml`.
- `talos_config` should derive etcd/control-plane members from `groups['hybrid'] + groups['controlplane']` and workers from `groups['worker']`.
- Dynamic inventories should produce groups named `hybrid`, `controlplane`, and `worker` from full role labels, not short codes.
- `docs/node-roles.md` defines three full role names: `controlplane`, `worker`, `hybrid`; short codes (`cp`, `wk`, `hb`) are only for generated hostnames.
- `docs/node-roles.md` states topology changes are declared by `hcloud_topology`, and the Hetzner dynamic inventory picks up new servers by cluster label after provisioning.
- `.ai/review-2026-06-10.md` identifies old coupling risks: `node_ips.yml` as cross-play filesystem IPC (M6) and Hetzner-coupled playbook/vars structure (M7).
- The review recommends this order for the portability refactor: vars split first, playbook rename second, dynamic inventory last.
- The review’s target contract is: provider provisioning produces node hostvars; Talos/Flux layers consume only `node_public_ip`, `node_private_ip`, role groups, and shared cluster identity.
- The review flags the dynamic inventory watch-outs: it needs a Hetzner token at inventory load time, hits the Hetzner API on inventory runs, is not useful before servers exist, and should consume actual private IPs while provisioning keeps deterministic private-IP assignment.

## Official docs to consult before implementation

Because this planning pass was intentionally limited to already-known repo context, implementation must consult official docs/README before code changes for these external interfaces:

- Ansible inventory plugin configuration and `compose`/`keyed_groups` behavior.
- `hetzner.hcloud.hcloud` dynamic inventory plugin docs, including valid inventory filename suffixes, token options, hostvars exposed, label filtering, and private-network IP hostvars.
- `hetzner.hcloud` collection module docs for any provisioning-label or network-attachment changes.
- Ansible variable precedence/loading docs for `inventories/common`, provider inventory directories, and `group_vars/all` vs provider-specific `group_vars`.
- `ansible-lint` docs only if any lint rule exceptions/noqa comments are added.

The implementing agent must cite the exact docs consulted in its handoff.

## Target shape

Use this target unless implementation-time inspection proves the repo has already converged:

```text
infra/ansible/
  inventories/
    common/group_vars/all/
      cluster.yml     # provider-neutral cluster identity, VIP/endpoint, node_role_short map
      talos.yml       # Talos version/extensions/schematic inputs
      flux.yml        # Flux repo/branch/path inputs
    providers/
      hcloud/
        hcloud.yml    # Hetzner dynamic inventory, label-filtered by cluster
        group_vars/all/
          hcloud.yml
          hcloud.sops.yaml
      manual/
        hosts.yml     # static nodes satisfying the same node_* hostvar contract
  playbooks/
    bootstrap-talos.yml
    bootstrap-flux.yml
    providers/hcloud/
      image-talos.yml
      provision-infra.yml
```

## Ordered implementation steps

### 0. Decision / ADR gate

- Owner: live session / task-planner; `adr-writer` only after user agreement.
- Depends on: none.
- Actions:
  - Confirm whether ADR notes A-D are already accepted as the durable architecture decision.
  - If not already recorded, pause before implementation and have `adr-writer` create/update an ADR under `docs/cluster-bootstrap/adrs/` covering the provider contract, layering, and dynamic inventory trade-offs.
- Acceptance criteria:
  - Either an existing agreed ADR is identified, or an ADR-writing follow-up is explicitly scheduled before implementation.
  - No implementor silently chooses a different provider seam.
- Validation:
  - Documentation review only; no commands.

### 1. Implementation preflight and docs confirmation

- Owner: `ansible-implementor`.
- Depends on: step 0.
- Actions:
  - Consult the official docs listed above before editing.
  - Inspect only the Ansible files necessary to map current names/vars/IP flow to the target shape.
  - Record whether each desired target is already present or still needs changes; skip no-op churn.
- Acceptance criteria:
  - Handoff includes official doc links consulted.
  - Handoff lists the exact files that will be changed and which review notes A-D each file addresses.
  - No code behavior is changed in this step.
- Validation commands:
  - None required beyond read-only inspection.

### 2. Split vars by layer/domain

- Owner: `ansible-implementor`.
- Depends on: step 1.
- Actions:
  - Move provider-neutral vars into common files by domain (`cluster.yml`, `talos.yml`, `flux.yml`) if not already split.
  - Move Hetzner-only vars into the hcloud provider inventory path, keeping the `hcloud_` prefix.
  - Keep secrets in `hcloud.sops.yaml`; do not decrypt or commit plaintext.
  - Preserve existing variable names unless a rename is essential for the inventory contract.
- Acceptance criteria:
  - Ansible still auto-loads all vars with `-i inventories/common -i inventories/providers/hcloud` and with `-i inventories/common -i inventories/providers/manual`.
  - `cluster_`, `talos_`, `flux_`, and `hcloud_` prefixes remain consistent.
  - Provider-neutral roles do not need provider-specific var filenames.
- Validation commands:
  - `cd infra/ansible && ansible-inventory -i inventories/common -i inventories/providers/manual --list`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check`

### 3. Rename/reorganize playbooks by concern and provider

- Owner: `ansible-implementor`.
- Depends on: step 2.
- Actions:
  - Put Hetzner-specific playbooks under `playbooks/providers/hcloud/`.
  - Use concern-oriented names for provider-neutral playbooks: `bootstrap-talos.yml` and `bootstrap-flux.yml`.
  - Update Ansible references/imports/includes affected by moves.
  - Do not change role behavior in the same step unless required by path resolution.
- Acceptance criteria:
  - Provider-specific playbooks can be identified by path without reading internals.
  - Provider-neutral playbooks do not reference hcloud-specific vars except through inventory-provided node hostvars.
  - Old playbook names are either removed or retained only as deliberate compatibility shims with a documented deprecation note.
- Validation commands:
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/image-talos.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/provision-infra.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check`

### 4. Make provider provisioning produce the inventory contract

- Owner: `ansible-implementor`.
- Depends on: step 3.
- Actions:
  - Ensure Hetzner server provisioning labels each node with full role names (`role=hybrid`, `role=controlplane`, `role=worker`) and cluster identity.
  - Ensure generated hostnames use `{cluster_name}-{short_code}{ordinal}` while labels/groups keep full role names.
  - Ensure deterministic private-IP assignment remains in provisioning inputs where required.
  - Ensure manual inventory examples/static hosts expose `node_role`, `node_public_ip`, and `node_private_ip` directly.
- Acceptance criteria:
  - A newly provisioned hcloud node can be discovered by `cluster=<cluster_name>` and grouped by full role label.
  - Manual inventory does not need any generated `node_ips.yml` file.
  - No provider-neutral role depends on Hetzner-specific labels or hostvars.
- Validation commands:
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/provision-infra.yml --syntax-check`
  - Optional/read-only API validation after credentials are available: `cd infra/ansible && ansible-inventory -i inventories/common -i inventories/providers/hcloud --graph`

### 5. Replace `node_ips.yml` with dynamic/static inventory hostvars

- Owner: `ansible-implementor`.
- Depends on: step 4.
- Actions:
  - Add or update hcloud dynamic inventory configuration so it filters by cluster label, builds role groups from `hcloud_labels.role`, and composes `node_public_ip` / `node_private_ip` from plugin hostvars.
  - Update `talos_config` and any related cluster bootstrap logic to read `hostvars[host].node_public_ip` and `hostvars[host].node_private_ip`.
  - Remove the old `node_ips.yml` write/read/include/default path once the inventory path is validated.
  - Keep token handling out of git; inventory token access must remain compatible with SOPS/module_defaults or an explicitly documented environment path.
- Acceptance criteria:
  - `bootstrap-talos.yml` can resolve all target nodes and required `node_*` hostvars from either hcloud dynamic inventory or manual static inventory.
  - No role fabricates or consumes `infra/talos/node_ips.yml` as cross-play IPC.
  - Hcloud inventory loading is read-only apart from normal Hetzner API queries.
  - Failure modes are clear when the hcloud token is unavailable during inventory loading.
- Validation commands:
  - `cd infra/ansible && ansible-inventory -i inventories/common -i inventories/providers/manual --list`
  - Read-only/API: `cd infra/ansible && ansible-inventory -i inventories/common -i inventories/providers/hcloud --list`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check`

### 6. Documentation updates for the new operator workflow

- Owner: `docs-writer`.
- Depends on: steps 2-5 passing syntax/inventory validation.
- Actions:
  - Update `docs/adding-a-provider.md` if implementation details differ from the target contract or command paths.
  - Update `docs/node-roles.md` if topology, labels, or generated hostnames changed.
  - Add/update human-facing command examples for hcloud and manual inventories.
  - Do not write ADR rationale here; if an ADR is needed, route to `adr-writer` after user agreement.
- Acceptance criteria:
  - Docs describe the actual implemented inventory contract and playbook paths.
  - Docs clearly distinguish provider-specific provisioning from provider-neutral bootstrap.
  - Docs mention the dynamic inventory token/API requirement and the no-SSH/Talos lockout boundary where relevant.
- Validation commands:
  - Documentation review plus link/path sanity check.
  - If markdown tooling exists in the repo, run it; otherwise no markdown command is required.

### 7. Final validation, review, and handoff

- Owner: `ansible-implementor` for command validation; `ansible-reviewer` for read-only review; `docs-writer` for docs fixes if reviewer finds docs drift.
- Depends on: steps 2-6.
- Actions:
  - Run safe local validation.
  - Run read-only hcloud inventory validation only with credentials available and with the user aware it contacts Hetzner API.
  - Do not run provisioning/bootstrap/upgrade against the live cluster unless the user explicitly approves.
  - Send Ansible changes to `ansible-reviewer`; loop fixes back to `ansible-implementor` until approved.
- Acceptance criteria:
  - `ansible-lint` passes or every remaining issue is explicitly documented with rationale.
  - Syntax checks pass for renamed playbooks.
  - Inventory validation proves both manual and hcloud inventory paths expose the node contract.
  - Reviewer approves idempotency, variable namespacing, secrets handling, and provider-neutral boundaries.
- Validation commands:
  - `cd infra/ansible && ansible-lint`
  - `cd infra/ansible && ansible-inventory -i inventories/common -i inventories/providers/manual --list`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/manual playbooks/bootstrap-talos.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/image-talos.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/provision-infra.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --syntax-check`
  - `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml --syntax-check`
  - Read-only/API if approved/credentialed: `cd infra/ansible && ansible-inventory -i inventories/common -i inventories/providers/hcloud --list`
  - Live-cluster check/diff only with explicit approval: `cd infra/ansible && ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml --check --diff`

## Risks, side effects, and live-cluster boundaries

- Dynamic hcloud inventory contacts the Hetzner API during inventory loading. This should be read-only and not cost-incurring, but it requires credentials earlier than module execution.
- Provisioning playbook changes can create/delete/relabel servers when run normally. The implementor must not run them against live infrastructure without explicit approval.
- Bootstrap playbook validation may contact Talos APIs if run beyond syntax/inventory checks. Treat `--check --diff` as live-cluster interaction and get approval when credentials/endpoints point at kube1.
- Removing `node_ips.yml` changes an operational fallback. Do not delete the old path until both hcloud dynamic inventory and manual static inventory resolve the required hostvars.
- Private IP source-of-truth must stay deterministic at attach/provision time even if Talos config consumes discovered actual IPs afterward.
- Role/group naming drift is high impact: full names belong in groups/labels/Kubernetes labels; short codes belong only in hostnames.
- Token handling is a security boundary. Do not move decrypted hcloud tokens into inventory files or committed plaintext.
- No `hcloud`, `talosctl`, `kubectl`, `flux reconcile`, provisioning, bootstrap, or upgrade commands should be run by automation in this refactor without explicit user approval.

## ADR needs

- ADR likely required if notes A-D are not already captured as an accepted decision, because the provider contract and dynamic inventory seam are durable bootstrap architecture.
- ADR location: `docs/cluster-bootstrap/adrs/` because the decision is provider-agnostic bootstrap architecture.
- ADR owner: `adr-writer`, after the user confirms the decision.
- The ADR should capture: context, decision, consequences, rejected alternatives (`node_ips.yml` IPC, provider-specific Talos roles, manual-only inventory), and trade-offs around portability, stale state, security/token loading, and live-cluster blast radius.
