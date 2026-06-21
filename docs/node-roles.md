# Node roles

## The three roles

| Role | Short code | Runs etcd | Schedules workloads | `allowSchedulingOnControlPlanes` |
|---|---|---|---|---|
| `controlplane` | `cp` | yes | no (taint) | false |
| `worker` | `wk` | no | yes | n/a |
| `hybrid` | `hb` | yes | yes | true |

**Why hybrid?** A 3-node cluster that is pure controlplane has no worker capacity. A
3-node cluster of pure workers has no Kubernetes API. Hybrid nodes do both — all three
nodes participate in etcd and all three accept workload pods. The trade-off is blast
radius: a bad deployment can saturate resources that etcd needs. For a homelab this is
acceptable; production clusters typically separate the roles once the node count permits.

## Naming convention

Hostnames follow `{cluster_name}-{short_code}{ordinal}`, e.g. `kube1-hb1`, `kube1-hb2`,
`kube1-hb3`. The short code is the only place abbreviations appear. Everywhere else —
Ansible group names, Hetzner server labels, Kubernetes node labels — the full name is
used:
- Ansible group: `hybrid`, `controlplane`, `worker`
- Hetzner label: `role: hybrid`
- Kubernetes label: `node-role.kubernetes.io/hybrid: ""`

The short code map lives in `inventories/common/group_vars/all/cluster.yml`:
```yaml
node_role_short:
  controlplane: cp
  hybrid: hb
  worker: wk
```

## How Kubernetes sees the role label

Kubernetes normally prevents non-system components from setting
`node-role.kubernetes.io/*` labels because they're in the restricted prefix and kubelet
ignores them. Talos applies `machine.nodeLabels` via its own node controller (which runs
as a system component), bypassing that restriction. The label lands on the Node object
after the node registers.

The `allowSchedulingOnControlPlanes: true` patch (hybrid nodes only) removes the
`node-role.kubernetes.io/control-plane:NoSchedule` taint that Talos sets by default on
controlplane nodes. Without it, the Kubernetes scheduler skips those nodes for regular
pods.

## Changing the topology

Topology is declared in `inventories/providers/hcloud/group_vars/all/hcloud.yml`:
```yaml
hcloud_topology:
  controlplane: 0
  hybrid: 3
  worker: 0
```

Changing counts and re-running `provision-infra.yml` creates or deletes servers to match.
The Hetzner dynamic inventory plugin then picks up the new servers automatically (it
queries by `cluster=kube1` label). Run `bootstrap-talos.yml` afterward to apply configs
to any new nodes.
