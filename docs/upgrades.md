# Talos upgrades

## Two upgrade paths

| Scenario | Playbook chain | What it does |
|---|---|---|
| Config-only change (no OS upgrade) | `render-config.yml` → `bootstrap-talos.yml` | Re-renders generated group_vars + Talos patches from `config/clusters/kube1/`, then re-applies machine config with `--mode no-reboot`; Talos applies live where possible |
| OS version or schematic change | `render-config.yml` → `image-talos.yml` (hcloud) → `upgrade-talos.yml` | Re-renders, builds a new snapshot if needed, then upgrades each node's OS one at a time with a health check between |

Always prefer `bootstrap-talos.yml` for config-only changes. Use `upgrade-talos.yml`
only when `talos.version` or `talos.extensions` has changed — it reboots each node
sequentially, which takes longer and has higher blast radius.

The render step (`playbooks/render-config.yml`) is the source of truth: it reads
the selected cluster's inputs under `config/clusters/kube1/` — `cluster.yaml`
(provider/features/versions/Flux repo), `nodes.yaml` (node list/roles), and the
Talos-shaped patch overlays under `config/clusters/kube1/talos/*` (per-role and
per-node) — and emits the generated `inventories/.../group_vars/` +
`infra/talos/patches/generated/` tree. Hand-editing generated files is a no-op —
the next render overwrites them. The legacy `config/overrides/` tree still exists
as a migration fallback but is no longer the primary source of truth.

## Upgrading the Talos version

1. Edit `talos.version` in `config/clusters/kube1/cluster.yaml`.
2. Re-render generated group_vars + Talos patches:
   ```bash
   cd infra/ansible
   ansible-playbook playbooks/render-config.yml
   ```
   Review the generated diff in git.
3. (hcloud) Build a new snapshot — `image-talos.yml` checks labels first and skips
   the build if a matching snapshot already exists:
   ```bash
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/image-talos.yml
   ```
   The snapshot is named `talos-<version>-<schematic-short>` and labelled with
   `talos-version` + `talos-schematic-id`. `provision-infra.yml` resolves the ID
   automatically from these labels — no manual copy needed.
4. Upgrade the running cluster:
   ```bash
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/upgrade-talos.yml
   ```
   This sequences nodes one at a time: re-applies config (with the new installer image
   URL baked in), upgrades, waits for `talosctl health` to pass, then moves to the next.

## Changing extensions (schematic change)

Same procedure as version upgrade — a new schematic ID means a new installer image URL,
which is treated as an OS-level change by Talos.

1. Edit `talos.extensions.base` (and/or `talos.extensions.extra`) in `config/clusters/kube1/cluster.yaml`.
2. Re-render and clear the schematic cache so it's re-resolved from factory.talos.dev:
   ```bash
   cd infra/ansible
   ansible-playbook playbooks/render-config.yml
   rm infra/talos/schematic_id
   ```
3. (hcloud) Build a new snapshot:
   ```bash
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/providers/hcloud/image-talos.yml
   ```
4. Upgrade the running cluster:
   ```bash
   ansible-playbook -i inventories/common -i inventories/providers/hcloud playbooks/upgrade-talos.yml
   ```

## Forcing a snapshot rebuild

`image-talos.yml` skips the build when a snapshot matching the current
`talos_version` + `talos_schematic_id` labels already exists in Hetzner Cloud.
To force a rebuild, delete the existing snapshot from the Hetzner Cloud console
(or via `hcloud image delete <id>`), then re-run `image-talos.yml`.

## Image convergence during bootstrap

`bootstrap-talos.yml` also converges the OS image — it reads the running installer image
on every node and upgrades any that differ from the factory target. This handles the case
where manual nodes booted from a generic Talos ISO and need to converge to the schematic.
On hcloud nodes that booted from the pre-built snapshot this is normally a no-op.

The difference from `upgrade-talos.yml`: bootstrap convergence runs all nodes in parallel
(no serial health-check between nodes). It's safe for initial bootstrap where there's no
live cluster to protect; for a live production-like cluster, use `upgrade-talos.yml`.

## What happens during a node upgrade

1. `talosctl upgrade` replicates the new installer image to the node.
2. Talos checks etcd quorum — it refuses if upgrading would break quorum (e.g. 2 of 3
   controlplane nodes already down).
3. Node reboots into the new image.
4. `talosctl health` polls until the node re-joins the cluster and etcd is healthy.
5. Next node starts.

Expect 2–4 minutes per node. The `--wait` flag on `talosctl upgrade` blocks until the
node is back; the health check adds a second gate before the next node proceeds.

## Rollback

Talos keeps the previous image on disk. To roll back a single node:
```bash
talosctl upgrade --nodes <IP> --talosconfig infra/talos/secrets/talosconfig \
  --image factory.talos.dev/installer/<previous-schematic-id>:<previous-version> --wait
```
The previous schematic ID is in git history (`config/clusters/kube1/cluster.yaml` for the version/extensions that produced it) or the Hetzner snapshot labels.
