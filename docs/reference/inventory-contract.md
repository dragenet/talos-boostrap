---
title: Inventory Contract
description: Required hostvars and group conventions that every provider inventory must supply for the bootstrap playbooks to work.
type: reference
audience: [operator, contributor]
tags: [ansible, inventory]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - playbooks.md
  - repository-layout.md
---

# Inventory Contract

The bootstrap playbooks (`bootstrap-talos.yml`, `bootstrap-flux.yml`, `upgrade-talos.yml`) are provider-agnostic. They never name a specific provider internally — they consume a standardized set of hostvars that every provider inventory must supply. This contract is defined by ADR-014.

---

## Required hostvars per node

Every host in the inventory must expose these variables:

| Hostvar | Type | Description |
|---|---|---|
| `node_public_ip` | string (IP) | Reachable IP address for the Talos API (`talosctl`, port `50000`) |
| `node_private_ip` | string (IP) | Private/cluster-internal IP used for VIP/etcd addressing |
| `node_role` | string | Full role label. One of: `controlplane`, `hybrid`, `worker` |

> **Note:** `node_private_ip` is a forward-looking contract field. On Hetzner Cloud it is derived from the API-assigned network IP. On providers without fixed-IP-honouring DHCP (e.g. bare-metal), it may need to match a static address configured in the Talos network patch.

---

## Node roles and Talos machine types

The `node_role` hostvar maps directly to a Talos machine type:

| `node_role` | Talos machine type | Function |
|---|---|---|
| `controlplane` | `controlplane` | Runs etcd, kube-apiserver, kube-controller-manager, kube-scheduler. No workload pods. |
| `hybrid` | `controlplane` | Runs the full control-plane stack **and** accepts workload pods (scheduler not tainted). Useful for small clusters where dedicated control-plane nodes would waste resources. |
| `worker` | `worker` | Runs workload pods only; no etcd or Kubernetes control-plane components. |

The short codes (`cp`, `hb`, `wk`) used in generated hostnames come from `cluster.nodeRoleShort` in `cluster.yaml` and are **not** used as group names in the inventory.

---

## Group conventions

Hosts must belong to the group that matches their `node_role`. The playbooks address nodes by group:

| Ansible group | Members |
|---|---|
| `controlplane` | Hosts with `node_role: controlplane` |
| `hybrid` | Hosts with `node_role: hybrid` |
| `worker` | Hosts with `node_role: worker` |

Both the dynamic (hcloud) and static (manual) inventories produce these groups using the same names so the playbooks need no provider-specific logic.

---

## How the inventories satisfy the contract

### Dynamic inventory (Hetzner Cloud)

The `hcloud` provider uses the `hetzner.hcloud.hcloud` dynamic inventory plugin (`inventories/providers/hcloud/hcloud.yml`). It queries the Hetzner API at inventory-load time and derives the required hostvars via `compose`:

```yaml
compose:
  node_public_ip: hcloud_ipv4
  node_private_ip: hcloud_private_ipv4
  node_role: hcloud_labels.role
```

Role groups are built with `keyed_groups` on the `role` server label:

```yaml
keyed_groups:
  - key: hcloud_labels.role
    prefix: ""
    separator: ""
```

Servers are filtered by `label_selector: cluster=<cluster_name>` so the inventory only sees nodes belonging to the target cluster. The `network:` setting ensures `hcloud_private_ipv4` is populated.

> **Chicken-and-egg note:** The dynamic inventory only sees servers that already exist, so it serves `bootstrap-talos.yml` and later playbooks — not `provision-infra.yml` (which runs against `localhost` and creates the servers).

### Static inventory (manual provider)

The `manual` provider uses a checked-in `inventories/providers/manual/hosts.yml`. The same hostvar names are declared directly on each host. Groups are defined explicitly:

```yaml
all:
  children:
    hybrid:
      vars:
        node_role: hybrid
      hosts:
        mycluster-hb1:
          node_public_ip: 1.2.3.4
          node_private_ip: 10.0.0.11
        mycluster-hb2:
          node_public_ip: 1.2.3.5
          node_private_ip: 10.0.0.12
```

---

## Running playbooks with an inventory

Bootstrap playbooks always require two inventory sources: `inventories/common` (shared group_vars and `localhost` definition) and the provider inventory:

```bash
# Hetzner Cloud
ansible-playbook \
  -i inventories/common \
  -i inventories/providers/hcloud \
  playbooks/bootstrap-talos.yml

# Manual / bare-metal
ansible-playbook \
  -i inventories/common \
  -i inventories/providers/manual \
  playbooks/bootstrap-talos.yml
```

`render-config.yml` runs only on `localhost` and does not need the provider inventory:

```bash
ansible-playbook playbooks/render-config.yml -e config_cluster=mycluster
```

---

## Adding a new provider

A new provider inventory satisfies the contract by exposing `node_public_ip`, `node_private_ip`, and `node_role` on every node host, and placing hosts in the correct `controlplane` / `hybrid` / `worker` groups. No changes to playbooks or roles are required.

See [Add a Provider](../guides/add-a-provider.md) for the full operational checklist.
