# Adding a provider

The cluster layer (`talos_config`, `talos_upgrade`, `flux_bootstrap`) is provider-neutral.
It consumes a contract of per-node hostvars and Ansible groups. Any provider that satisfies
the contract can drive the cluster playbooks without modification.

## The inventory contract

Every node must expose these hostvars:

| hostvar | type | description |
|---|---|---|
| `node_role` | string | `controlplane`, `worker`, or `hybrid` |
| `node_public_ip` | string | IP reachable by talosctl from your admin machine |
| `node_private_ip` | string | IP on the cluster's private network (forward-looking — see note below) |

> **`node_private_ip` is currently a forward-looking contract field.** The hcloud dynamic
> inventory populates it from `hcloud_private_ipv4`, but no role consumes it yet — Talos
> uses DHCP on the private interface plus a static VIP, which converges with Hetzner's
> fixed API-level IP assignment. New providers should still supply it; a future role may
> need it.

The inventory hostname is used as the Talos / Kubernetes node name. For generated
providers (hcloud) it follows `{cluster_name}-{short_code}{n}`. For manual inventories,
choose names that are DNS-label-safe.

Additionally:
- `ansible_connection: local` — all cluster commands run locally via talosctl; no SSH
  to cluster nodes.
- `ansible_python_interpreter: "{{ ansible_playbook_python }}"` — avoids version
  mismatches. Both are already set in `inventories/common/group_vars/all/cluster.yml`.

## Ansible groups

The `talos_config` role reads nodes from two computed lists (not from Ansible groups
directly, but groups power them):
- `groups['hybrid'] + groups['controlplane']` → etcd members (receive controlplane.yaml)
- `groups['worker']` → workers (receive worker.yaml)

For static inventories (`hosts.yml`), declare membership explicitly:
```yaml
all:
  children:
    hybrid:
      hosts:
        example-hb1:
          node_role: hybrid
          node_public_ip: 1.2.3.4
          node_private_ip: 10.0.0.2
```

For dynamic inventories (like hcloud), use `keyed_groups` on the `role` label to derive
Ansible groups, and `compose` to map provider-specific hostvars to the contract hostvars:
```yaml
keyed_groups:
  - key: hcloud_labels.role
    prefix: ""
    separator: ""
compose:
  node_public_ip: hcloud_ipv4
  node_private_ip: hcloud_private_ipv4
  node_role: hcloud_labels.role
```
`keyed_groups` produces Ansible groups named `hybrid`, `controlplane`, `worker` matching
the label values. `compose` sets the contract hostvars from provider-specific values —
without it, the groups exist but `node_public_ip`, `node_private_ip`, and `node_role`
are unset and the cluster playbooks fail. See ADR-014
(`docs/cluster-bootstrap/adrs/014-node-discovery-dynamic-inventory.md`) for the rationale
behind dynamic inventory over static host lists.

## Step-by-step: adding a new provider

> **`group_vars/all/<provider>.yml` is render-generated** by
> `playbooks/render-config.yml`. Provider-specific vars (region, server type, etc.)
> live under the provider key in `config/clusters/<cluster>/cluster.yaml` (e.g. `hcloud: ...`).
> The render compiler reads that key and emits the provider's `group_vars` file. Don't
> hand-edit the generated file — the next render overwrites it.

1. Create `inventories/providers/<provider>/` with:
   - `<provider>.yml` — dynamic plugin config, or `hosts.yml` — static host list
   - `group_vars/all/<provider>.sops.yaml` (if the provider needs an API token)
   - Provider-specific values are added to `config/clusters/<cluster>/cluster.yaml` (or
     `config/defaults/cluster.yaml` for provider-agnostic defaults) — the renderer
     emits `group_vars/all/<provider>.yml` from them.

   > **Dynamic inventories need API credentials at load time.** The inventory plugin
   > runs before `group_vars` are loaded, so it cannot read `group_vars` for
   > authentication. For hcloud, the plugin config uses
   > `lookup('community.sops.sops', ...)` to read the token directly from the
   > SOPS-encrypted secrets file. Your provider's plugin config will need a similar
   > mechanism — a `lookup()` plugin, an env var, or a credential file — whatever
   > works at inventory-load time. Static inventories (`hosts.yml`) have no such
   > requirement.

2. No `.sops.yaml` edit needed — the existing generic regex
   (`infra/ansible/inventories/.*/group_vars/.*\.sops\.ya?ml$`) already matches any new
   provider path. Only edit `.sops.yaml` if you need to add a new age recipient.

3. Write a provisioning playbook at `playbooks/providers/<provider>/provision-infra.yml`. It must
   produce nodes with the contract hostvars above and label them so the dynamic inventory
   picks them up.

4. Write a disk image playbook at `playbooks/providers/<provider>/image-talos.yml` if your provider
   needs a pre-built image. Call the `talos_schematic` role first — it sets
   `talos_disk_image_url` which you use to download and write the image.

5. Run the cluster playbooks with the new inventory:
   ```bash
   ansible-playbook -i inventories/common -i inventories/providers/<provider> playbooks/bootstrap-talos.yml
   ```

   > **Ordering matters: provision first, then bootstrap.** A dynamic inventory only
   > sees servers that already exist — before `provision-infra.yml` creates them, the
   > inventory returns an empty host list and the cluster playbooks have nothing to do.
   > Always run the provisioning playbook before the cluster playbooks when building
   > from scratch.

No changes to `talos_config`, `talos_upgrade`, or `flux_bootstrap` are needed.

## The hcloud provider as reference

`inventories/providers/hcloud/` and `playbooks/providers/hcloud/` are the canonical example. Key patterns:
- `hcloud_topology` count map (in the generated `group_vars/all/hcloud.yml`, sourced
  from `config/clusters/<cluster>/cluster.yaml` `hcloud.topology`) drives server creation — no
  static host list needed.
- Server labels (`cluster=example`, `role=hybrid`) are the bridge between provisioning and
  the dynamic inventory plugin.
- `hcloud.sops.yaml` holds the API token (user/secret-owned, never rendered); the
  `module_defaults` group key `group/hetzner.hcloud.all: { api_token: ... }` injects
  it into every hcloud task.
- The snapshot name `talos-{version}-{schematic_short}` encodes what's baked in, so
  you can tell at a glance which snapshot matches which config.
