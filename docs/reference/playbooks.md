---
title: Ansible Playbooks
description: Reference for all Ansible playbooks — what each does, when to run it, and what prerequisites it requires.
type: reference
audience: [operator, user]
tags: [ansible, playbooks]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - inventory-contract.md
  - cluster-yaml-schema.md
---

# Ansible Playbooks

All playbooks live under `infra/ansible/playbooks/`. Run them from the `infra/ansible/` directory or pass `-i` paths relative to that directory.

Provider-specific playbooks are under `playbooks/providers/<provider>/` and are only needed for that provider.

---

## Playbook summary

| Playbook | Purpose | When to run | Notes |
|---|---|---|---|
| `render-config.yml` | Renders Talos machine configs and Ansible group_vars from `cluster.yaml` + overlays | Before any other playbook; re-run after any `cluster.yaml` change | Runs on localhost only; no API calls outside the repo tree |
| `bootstrap-talos.yml` | Applies machine configs to nodes, bootstraps etcd, fetches kubeconfig | After provisioning nodes; re-run to push config changes to a running cluster | Requires inventory with `node_public_ip`, `node_private_ip`, `node_role` per node |
| `bootstrap-flux.yml` | Installs pre-Flux components (Cilium, optionally hcloud-ccm) and bootstraps FluxCD | After `bootstrap-talos.yml` succeeds and kubeconfig is available | Requires `flux_repo_url`; requires `hcloud_token` when CCM is enabled |
| `upgrade-talos.yml` | Re-applies machine configs then upgrades Talos OS one node at a time | When `talos.version` or `talos.extensions` changes | Each node reboots; Talos enforces etcd quorum safety automatically |
| `providers/hcloud/image-talos.yml` | Builds a Talos OS snapshot on Hetzner Cloud via rescue mode | Before `provision-infra.yml`; re-run after changing `talos.version` or `talos.extensions` | Hetzner Cloud only; idempotent — skips build if matching snapshot already exists |
| `providers/hcloud/provision-infra.yml` | Provisions Hetzner Cloud infrastructure (network, firewall, servers) | Once per cluster; re-run to converge infrastructure to declared state | Hetzner Cloud only; requires a Talos snapshot built by `image-talos.yml` |

---

## Detailed reference

### `render-config.yml`

Reads `config/defaults/cluster.yaml` and `config/clusters/<cluster>/cluster.yaml` (plus optional `nodes.yaml`), deep-merges them (cluster config wins per-key), validates, and emits:

- Generated `group_vars` consumed by all other playbooks
- Talos machine-config patch files under `infra/talos/patches/generated/`

**Runs on:** `localhost` only — no remote connections, no API calls outside the repo.

**Key variables:**

| Variable | Default | Description |
|---|---|---|
| `config_cluster` | `example` | Selects which `config/clusters/<name>/` directory to load |
| `config_render_check` | `false` | When `true`, renders to a temp dir and diffs against committed files (drift detection) |

**Usage:**
```bash
ansible-playbook playbooks/render-config.yml
ansible-playbook playbooks/render-config.yml -e config_cluster=mycluster
ansible-playbook playbooks/render-config.yml -e config_render_check=true
```

---

### `bootstrap-talos.yml`

Applies Talos machine configs to all cluster nodes and bootstraps etcd on first run. Safe to re-run — re-applies config changes to running nodes without reprovisioning.

**What it does:**
1. Resolves the Talos schematic (extensions → installer image URL via `factory.talos.dev`)
2. Generates `controlplane.yaml`, `worker.yaml`, and `talosconfig` (skipped if they already exist; delete to regenerate)
3. Applies machine configs to nodes in maintenance mode (first bootstrap) or running nodes (config updates)
4. Bootstraps etcd (idempotent — no-op if already bootstrapped)
5. Waits for cluster health and fetches `kubeconfig`
6. Optionally patches the kubeconfig server URL for provider-specific external access

**Prerequisites:**
- `render-config.yml` has been run
- Nodes are reachable at `node_public_ip` on Talos API port `50000`
- Inventory exposes `node_public_ip`, `node_private_ip`, `node_role` per node

**Usage:**
```bash
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-talos.yml
ansible-playbook -i inventories/common -i inventories/providers/manual  playbooks/bootstrap-talos.yml
```

> **Note:** Use `upgrade-talos.yml` instead when upgrading `talos_version` or the extension set on a live cluster — it sequences one node at a time with health checks between each.

---

### `bootstrap-flux.yml`

Installs pre-Flux cluster components and bootstraps FluxCD from the configured git repository.

**What it does:**
1. Pre-flight checks: verifies `kubectl`, `helm`, `flux` CLIs are available; asserts required variables
2. Ensures required namespaces exist (e.g. `cilium-system`)
3. Installs/upgrades Cilium via Helm with Talos-compatible values (KubePrism endpoint, BPF settings)
4. Installs hcloud-ccm (only when `hcloud-ccm` is in `bootstrap_pre_flux_components`)
5. Runs `flux bootstrap` with the configured git provider, repo URL, branch, and path

**Prerequisites:**
- `bootstrap-talos.yml` has completed successfully
- `infra/talos/secrets/kubeconfig` exists
- `flux_repo_url` is set (from rendered group_vars)
- `hcloud_token` is available when hcloud-ccm is required

**Usage:**
```bash
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/bootstrap-flux.yml
ansible-playbook -i inventories/common -i inventories/providers/manual  playbooks/bootstrap-flux.yml
```

---

### `upgrade-talos.yml`

Upgrades the Talos OS on all nodes to the configured version and schematic. Also re-applies machine configs so non-OS config changes (e.g. `kubernetes_version`, extension list, `nodeLabels`) land atomically with the OS upgrade.

**What it does:**
1. Resolves the Talos schematic (new installer image URL)
2. Re-applies machine configs to all running nodes
3. Upgrades each node's OS **one at a time**: reads the running image, runs `talosctl upgrade` only if the image differs, waits for cluster health before proceeding to the next node
4. Nodes already running the target image are skipped for the upgrade step (config is still re-applied)

**Prerequisites:**
- Cluster is healthy and bootstrapped
- Inventory is populated (nodes exist and are reachable)

**Usage:**
```bash
ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/upgrade-talos.yml
ansible-playbook -i inventories/common -i inventories/providers/manual  playbooks/upgrade-talos.yml
```

> **Note:** Each upgraded node reboots. Talos enforces etcd quorum safety and refuses to upgrade a control-plane node if doing so would break quorum.

---

### `providers/hcloud/image-talos.yml` _(Hetzner Cloud only)_

Builds a Talos OS snapshot on Hetzner Cloud by booting a temporary server into rescue mode, writing the Talos disk image, snapshotting the disk, and deleting the builder.

**Idempotency:** Queries Hetzner for an existing snapshot with matching `talos-version` and `talos-schematic-id` labels. Skips the build if one is found. Delete the snapshot in the Hetzner console to force a rebuild.

**Prerequisites:**
- `hcloud_token` set in `inventories/providers/hcloud/group_vars/all/hcloud.sops.yaml`
- SSH key registered in Hetzner Cloud (`hcloud.sshKeyName` in cluster config)
- `hcloud_builder_server_types` configured (set via rendered group_vars from `hcloud.builderServerTypes`)

**Usage:**
```bash
ansible-playbook -i inventories/common -i inventories/providers/hcloud \
  playbooks/providers/hcloud/image-talos.yml
```

---

### `providers/hcloud/provision-infra.yml` _(Hetzner Cloud only)_

Provisions Hetzner Cloud infrastructure: private network, subnet, firewall, placement group, and servers. Idempotent — re-running converges to the declared state without recreating existing resources.

**Resolves the Talos snapshot dynamically** by querying Hetzner for a snapshot whose labels match the current `talos_version` + `talos_schematic_id`. Fails fast with a helpful message if no matching snapshot is found.

**Prerequisites:**
- `image-talos.yml` has been run at least once for the current `talos_version` + extensions
- `hcloud_token` is available
- `hcloud.*` variables are configured in cluster config

**Usage:**
```bash
ansible-playbook -i inventories/common -i inventories/providers/hcloud \
  playbooks/providers/hcloud/provision-infra.yml
```
