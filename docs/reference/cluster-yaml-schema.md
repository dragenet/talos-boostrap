---
title: cluster.yaml Schema
description: Full field reference for the cluster.yaml configuration input file — types, defaults, and a minimal example.
type: reference
audience: [user, operator]
tags: [config, schema]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - playbooks.md
  - repository-layout.md
---

# cluster.yaml Schema

`cluster.yaml` is the primary configuration input for a cluster. It is a YAML file validated against `config/schemas/cluster-input.schema.json`.

## Location

```
config/clusters/<cluster-name>/cluster.yaml
```

The render deep-merges `config/defaults/cluster.yaml` (template-owned defaults) with `config/clusters/<cluster>/cluster.yaml` (user-owned overrides). The user file wins per-key. Users edit only their cluster file; `config/defaults/cluster.yaml` is template-owned and must not be modified.

After editing, re-run the config compiler to regenerate artifacts:

```bash
ansible-playbook playbooks/render-config.yml -e config_cluster=<cluster-name>
```

---

## Top-level sections

### `cluster`

Core cluster identity and API settings.

| Field | Type | Default | Description |
|---|---|---|---|
| `cluster.name` | string | `""` | Cluster name; used in generated hostnames and Talos cluster identity. **Required.** |
| `cluster.kubernetesVersion` | string | `v1.32.0` | Kubernetes version to deploy and maintain. |
| `cluster.vip` | string | `""` | Virtual IP for the Talos/Kubernetes API. Single source of truth — do not set `talos.network.vip`. **Required.** |
| `cluster.endpoint` | string | `https://{{ cluster_vip }}:6443` | Kubernetes API endpoint. Computed from `vip` by default; override only if the API is reached via a different address (e.g. external load balancer). |
| `cluster.certSANs.talos` | list[string] | `[]` | Extra SANs added to Talos machine TLS certificates (`machine.certSANs`). |
| `cluster.certSANs.kubernetes` | list[string] | `[]` | Extra SANs added to the kube-apiserver certificate (`cluster.apiServer.certSANs`). |
| `cluster.nodeRoleShort` | map | `{controlplane: cp, hybrid: hb, worker: wk}` | Short codes used in generated hostnames only (e.g. `mycluster-cp1`). |

---

### `provider`

Selects the infrastructure provider.

| Field | Type | Default | Description |
|---|---|---|---|
| `provider.name` | string | `manual` | Provider identifier. Supported values: `hcloud`, `manual`. Determines which provider group_vars are rendered and which provider-gated features are available. |

---

### `flux`

FluxCD bootstrap configuration.

| Field | Type | Default | Description |
|---|---|---|---|
| `flux.provider` | string | `generic` | Git provider for bootstrap. One of: `generic`, `github`, `bitbucket-server`. |
| `flux.repoUrl` | string | `""` | HTTPS or SSH repository URL. Required for `generic` git flows. |
| `flux.repoBranch` | string | `main` | Git branch Flux tracks. |
| `flux.path` | string | `""` | Path within the repository where Flux manifests live (e.g. `./flux/clusters/mycluster`). |
| `flux.auth.mode` | string | `ssh-private-key` | Authentication mode. One of: `provider-deploy-key`, `https-token`, `ssh-private-key`. |
| `flux.auth.username` | string | `git` | HTTPS basic-auth username. |
| `flux.auth.owner` | string | `""` | Repository owner (GitHub org/user or Bitbucket project key). Required for `github` and `bitbucket-server` providers. |
| `flux.auth.repository` | string | `""` | Repository slug/name. Required for `github` and `bitbucket-server` providers. |
| `flux.auth.hostname` | string | `""` | Custom hostname for GitHub Enterprise or Bitbucket Server/DC. Omit for standard cloud-hosted providers. |
| `flux.auth.personal` | bool | `false` | Use personal account instead of organisation (GitHub only). |
| `flux.auth.private` | bool | `true` | Create a private repository (provider-specific bootstrap). |
| `flux.auth.readWriteKey` | bool | `false` | Request a read-write deploy key (provider-deploy-key mode only). |
| `flux.auth.tokenAuthBearer` | bool | `false` | Use bearer token instead of HTTPS basic auth for generic git flows. |
| `flux.auth.sshHostname` | string | `""` | Non-standard SSH host:port (e.g. for Bitbucket Server/DC). Maps to `--ssh-hostname`. |
| `flux.auth.privateKeyFile` | string | `""` | Local path to SSH private key file for `ssh-private-key` mode. Not committed to git. |
| `flux.auth.caFile` | string | `""` | Custom CA certificate file for HTTPS connections. |
| `flux.auth.allowInsecureHttp` | bool | `false` | Allow insecure HTTP connections (development/testing only). |
| `flux.auth.silent` | bool | `false` | Assume deploy key is already set up (provider-deploy-key mode). |

---

### `talos`

Talos Linux version and node configuration.

| Field | Type | Default | Description |
|---|---|---|---|
| `talos.version` | string | `v1.13.3` | Talos Linux version to install and maintain. |
| `talos.extensions.base` | list[string] | `[]` | Base system extensions included in every image (e.g. `siderolabs/qemu-guest-agent`). |
| `talos.extensions.extra` | list[string] | `[]` | Additional user-specified extensions merged with base and feature-implied extensions. |
| `talos.network.defaultInterface.interface` | string\|null | `null` | Network interface name for the primary interface (fallback; prefer `deviceSelector`). |
| `talos.network.defaultInterface.deviceSelector` | map\|null | `null` | Hardware-stable device selector (e.g. `{hardwareAddr: "86:00:00:*"}`). Preferred over `interface`. |

> **Important:** Do not set `talos.network.vip`. The renderer reads `cluster.vip` as the single source of truth and rejects `talos.network.vip` in overrides to prevent silent divergence.

---

### `cni`

CNI plugin selection.

| Field | Type | Default | Description |
|---|---|---|---|
| `cni.name` | string | `cilium` | CNI plugin. One of: `cilium`, `none`. `cilium` disables in-cluster CNI management for pre-Flux Cilium install. `none` leaves Talos default in-cluster CNI management enabled. |

---

### `cloud_controller_manager`

External Cloud Controller Manager configuration.

| Field | Type | Default | Description |
|---|---|---|---|
| `cloud_controller_manager.enabled` | string\|bool | `auto` | CCM tri-state: `auto` (resolve per provider — enabled for `hcloud`, disabled for `manual`), `true` (force-enable; fails if provider has no supported CCM), `false` (disable). |

---

### `features`

Optional cluster add-ons deployed by Flux after bootstrap.

| Field | Type | Default | Description |
|---|---|---|---|
| `features.certManager` | bool | `false` | Deploy cert-manager via Flux. |
| `features.ingress` | string | `none` | Ingress controller. One of: `none`, `envoy-gateway`, `traefik`. |
| `features.openebs` | bool | `false` | Deploy OpenEBS LocalLV via Flux. |
| `features.hcloudCsi` | bool | `false` | Deploy Hetzner Cloud CSI driver via Flux. |

---

### `ingress`

Gateway API CRD configuration.

| Field | Type | Default | Description |
|---|---|---|---|
| `ingress.gatewayApiChannel` | string | `standard` | Gateway API CRD channel. One of: `standard`, `experimental`. |

---

### `storage`

Talos volume configuration (ADR-013).

| Field | Type | Default | Description |
|---|---|---|---|
| `storage.volumes` | list[Volume] | `[]` | List of storage volume definitions. User-owned list replaces the default. |

**Volume object fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Volume name |
| `type` | string | Yes | Volume type (e.g. `raw`) |
| `diskSelector` | string | Yes | Disk selector (e.g. `system_disk`) |
| `minSize` | string | Yes | Minimum size (e.g. `20GB`) |
| `maxSize` | string | Yes | Maximum size (e.g. `20GB`) |
| `openebs` | bool | No | Mark volume as the OpenEBS LocalLV backing device |

> **Note:** LUKS/encryption keys are structurally rejected by the schema (`additionalProperties: false`) due to an unresolved Talos shutdown/upgrade deadlock with LUKS+LVM (talos#13354).

---

### `hcloud`

Hetzner Cloud provider settings. Only used when `provider.name: hcloud`.

| Field | Type | Default | Description |
|---|---|---|---|
| `hcloud.location` | string | `nbg1` | Hetzner Cloud location (datacenter region). |
| `hcloud.serverTypes` | list[string] | `[]` | Prioritized server type list; first available in location wins. |
| `hcloud.serverTypesPerRole` | map | `{}` | Per-role server type overrides (keys: `controlplane`, `hybrid`, `worker`). |
| `hcloud.builderServerTypes` | list[string] | `[]` | Server types for the temporary image builder (small types are fine). |
| `hcloud.privateNetworkName` | string | `""` | Name of the Hetzner Cloud private network to create/use. |
| `hcloud.privateNetworkCidr` | string | `10.0.0.0/16` | CIDR for the private network. |
| `hcloud.privateSubnetCidr` | string | `10.0.0.0/24` | CIDR for the private subnet. |
| `hcloud.firewallName` | string | `""` | Name of the Hetzner Cloud firewall to create/use. |
| `hcloud.sshKeyName` | string | `""` | SSH key name registered in Hetzner Cloud (used during image build). |
| `hcloud.firewallOpenAccess` | bool | `false` | Open API ports to `0.0.0.0/0` (true) or scope to `adminIps` (false). |
| `hcloud.adminIps` | list[string] | `[]` | CIDRs allowed to reach the Talos and Kubernetes APIs. Empty list auto-detects current public IP at provision time. |
| `hcloud.topology.controlplane` | int | `0` | Number of control-plane nodes to provision. |
| `hcloud.topology.hybrid` | int | `0` | Number of hybrid nodes (control-plane + worker combined) to provision. |
| `hcloud.topology.worker` | int | `0` | Number of worker-only nodes to provision. |
| `hcloud.serversIpOffset` | int | `11` | Starting octet for node private IPs within the subnet (e.g. `11` → `.11`, `.12`, …). |

---

## Minimal example

The following is the minimum viable `cluster.yaml` for a manual-provider cluster:

```yaml
cluster:
  name: mycluster
  kubernetesVersion: v1.32.0
  vip: 192.168.1.10

provider:
  name: manual

flux:
  provider: generic
  repoUrl: ssh://git@github.com/myorg/myrepo.git
  repoBranch: main
  path: ./flux/clusters/mycluster
  auth:
    mode: ssh-private-key
    privateKeyFile: ~/.ssh/flux_deploy_key

talos:
  version: v1.13.3
  network:
    defaultInterface:
      interface: eth0
```
