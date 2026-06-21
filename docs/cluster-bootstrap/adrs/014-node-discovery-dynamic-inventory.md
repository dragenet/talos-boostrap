# ADR-014: Node discovery via inventory, not filesystem IPC

**Status:** Accepted

## Context

The bootstrap layer needs, per node: a reachable Talos API address, a private IP (for VIP/etcd), a role (`controlplane`/`hybrid`/`worker`), and shared cluster identity. The roles themselves (`talos_config`, `talos_upgrade`) are provider-agnostic — only node-IP ingestion was the coupling point.

The original design used a hand-written `infra/talos/node_ips.yml` as cross-play filesystem IPC: the `hcloud_servers` provisioning role wrote it, and `talos_config` read it back via `include_vars`. This was Hetzner-shaped, non-idempotent across separate `ansible-playbook` invocations, and forced every alternative provider to fabricate the same file — a direct violation of the provider-agnostic seam (ADR-007).

## Decision

**Node discovery is owned by inventory, not by filesystem IPC.** The `node_ips.yml` file is removed; node addresses and roles flow through standard Ansible hostvars produced by the inventory mechanism.

1. **Dynamic inventory for hcloud, static hostvars for manual.** The `hetzner.hcloud.hcloud` dynamic inventory plugin discovers live servers; the `manual` provider uses a checked-in `hosts.yml` with the same hostvar contract. Both produce identical variable names for the roles to consume.

2. **Standardized hostvar contract.** Every provider declares: `node_public_ip`, `node_private_ip`, `node_role`, plus membership in role groups (`controlplane`, `hybrid`, `worker`) built from full role labels — short codes appear only in generated hostnames (per `docs/node-roles.md`). Provider-agnostic roles read `hostvars[node].node_public_ip` / `.node_private_ip` and never name a provider.

3. **hcloud plugin config.** `label_selector: cluster=<cluster_name>` scopes to kube1 nodes; `keyed_groups` on `hcloud_labels.role` builds the role groups; `compose` derives `node_public_ip` / `node_private_ip` / `node_role` from plugin hostvars. `network:` is set to the private network name so `hcloud_private_ipv4` (and thus `node_private_ip`) is reliably populated.

4. **DHCP on the Talos side, not static per-node IPs.** The Talos machine config uses `dhcp: true` on the private interface plus a static VIP (`cluster_vip`). On Hetzner, the provider assigns a fixed private IP at the API level (`server_network.ip`) and its DHCP serves that exact IP, so DHCP and static converge. `node_private_ip` is therefore a **forward-looking contract field** — declared by every provider, consumed only by providers that lack a fixed-IP-honouring DHCP (e.g. bare-metal `manual` may need static `addresses:` in future). This keeps the Talos config a single shared per-role patch rather than a per-node template.

## Consequences

- The dynamic inventory contacts the Hetzner API at inventory-load time and needs the hcloud token available to the inventory plugin (via SOPS lookup, same secret as `module_defaults`) — a different code path than playbook `module_defaults`; both must keep working.
- **Chicken-and-egg by design.** Dynamic inventory only sees servers that already exist, so it serves `bootstrap-talos` / `bootstrap-flux`, not `provision-infra` / `image-talos` (those run against `localhost`). This matches the natural provision → bootstrap split.
- `network:` filters the inventory to network-attached servers; all kube1 nodes are attached before bootstrap runs, so this is safe — but it means inventory is empty until provisioning completes.
- `node_private_ip` being declared-but-currently-unconsumed on Hetzner is intentional, not a gap. It exists so providers without fixed-IP DHCP can wire static addresses later without changing the contract.
- Private-IP source-of-truth stays deterministic at provisioning time (`hcloud_servers` computes `subnet_base + offset + index` and passes it to `server_network.ip`); the inventory *discovers* the actual assigned IP afterward. VIP/etcd addressing remains deterministic.
- Rejected alternatives, folded into the rationale: the old `node_ips.yml` filesystem IPC (non-idiomatic, stale-state-prone, Hetzner-shaped); manual-only / no dynamic inventory (loses live IP discovery, recreated servers desync a file on disk); provider-specific Talos roles (would fork `talos_config` per provider, breaking ADR-007); static per-node private IPs in the Talos config on Hetzner (would require per-node config patches, breaking the single shared `controlplane.yaml.j2` / `worker.yaml.j2` template model, for zero practical benefit since Hetzner DHCP already yields the fixed API-assigned address — kept as a future option for providers that need it, which is exactly why `node_private_ip` stays in the contract).
- Cross-references: implements the inventory-mechanism half of ADR-007 (portable bootstrap structure); complements ADR-010 (Ansible→Flux handoff). The operational contract for new providers is documented in `docs/adding-a-provider.md`.
