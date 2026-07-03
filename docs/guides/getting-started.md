---
title: Getting Started
description: End-to-end first-run guide — from prerequisites to a running Talos Kubernetes cluster managed by Flux.
type: guide
audience: [user, operator]
tags: [getting-started, bootstrap, talos, flux]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - configure-your-cluster.md
  - gitops-with-flux.md
  - ../concepts/architecture-overview.md
---

# Getting Started

This guide walks you through bootstrapping a new Talos Linux Kubernetes cluster from scratch using this template — from checking prerequisites to a running cluster with Flux reconciling your GitOps tree.

The bootstrap sequence has four phases:

1. **Render configuration** — compile cluster inputs into Ansible `group_vars`, Talos patches, and Flux selections.
2. **Provision infrastructure** — create cloud or physical nodes (provider-specific).
3. **Bootstrap Talos** — apply machine configs, bootstrap etcd, and fetch `kubeconfig`.
4. **Bootstrap Flux** — install pre-Flux components (CNI, CCM) and run `flux bootstrap`.

All Ansible commands in this guide are run from `infra/ansible/` unless noted otherwise.

---

## Prerequisites

### Tools

Install the following tools on your control machine before proceeding:

| Tool | Purpose |
|---|---|
| `talosctl` | Interact with Talos nodes |
| `kubectl` | Interact with the Kubernetes API |
| `ansible` (ansible-core ≥ 2.15) | Run provisioning and bootstrap playbooks |
| `flux` CLI | Bootstrap and monitor FluxCD |
| Provider CLI | Cloud provider control (use your provider's CLI) |
| `sops` + `age` | Decrypt SOPS-encrypted secrets used by Ansible |
| `helm` | Required by Ansible roles that install pre-Flux Helm releases |

Install the required Ansible collections from `infra/ansible/`:

```shell
ansible-galaxy collection install -r requirements.yml
```

This installs:
- `hetzner.hcloud` — Hetzner Cloud modules (provider-gated)
- `community.sops` — SOPS vars plugin for automatic secret decryption
- `kubernetes.core` — Kubernetes and Helm modules used by the Flux bootstrap role

The `kubernetes.core` collection also requires the `kubernetes` Python package:

```shell
pip install kubernetes
```

### Repository

Clone this repository and treat it as the canonical source for your cluster:

```shell
git clone <your-repo-url> <cluster-repo>
cd <cluster-repo>
```

This is a template repository. You own everything under `config/clusters/<cluster-name>/`; template updates land in `config/defaults/` and playbooks, which you can pull without conflicting with your cluster config.

---

## Step 1 — Create Your Cluster Config

Copy the `example` cluster config as a starting point:

```shell
cp -r config/clusters/example config/clusters/<cluster-name>
```

Edit `config/clusters/<cluster-name>/cluster.yaml` with your cluster's values — at minimum:

- `cluster.name` — a short identifier
- `cluster.vip` — the floating IP for the Kubernetes API
- `provider.name` — `hcloud` or `manual`
- `flux.repoUrl` and `flux.path`

See [Configure Your Cluster](configure-your-cluster.md) for a full walkthrough of configuration options.

---

## Step 2 — Render Configuration

The render playbook compiles your inputs into all generated artifacts: Ansible `group_vars`, Talos machine-config patches, and Flux kustomization selections.

```shell
# From infra/ansible/
ansible-playbook playbooks/render-config.yml -e config_cluster=<cluster-name>
```

The render runs on `localhost`, contacts no external APIs, and is safe to re-run. Review the generated diffs before proceeding.

> **Drift invariant:** CI asserts that `render(inputs) == committed tree`. Commit the generated output before running live playbooks so the repo remains self-describing.

---

## Step 3 — Provision Infrastructure

This step is provider-specific. If you are using a managed cloud provider, run its provisioning playbook to create the nodes. For manually provisioned hardware, boot the nodes from a Talos image and skip this step.

**Example — `<provider>`:**

```shell
# From infra/ansible/
ansible-playbook \
  -i inventories/common \
  -i inventories/providers/<provider> \
  playbooks/providers/<provider>/provision-infra.yml
```

For other providers, run the equivalent playbook under `playbooks/providers/<provider>/`.

---

## Step 4 — Bootstrap Talos

Apply Talos machine configs to all nodes, bootstrap etcd, and retrieve `kubeconfig`.

```shell
# From infra/ansible/
ansible-playbook \
  -i inventories/common \
  -i inventories/providers/<provider> \
  playbooks/bootstrap-talos.yml
```

This playbook:
- Resolves the Talos schematic (extensions → installer image URL)
- Generates `controlplane.yaml`, `worker.yaml`, and `talosconfig` (once; delete to regenerate)
- Applies configs to nodes in maintenance mode (first bootstrap) or re-applies to running nodes
- Bootstraps etcd (idempotent — no-op if already done)
- Waits for cluster health and fetches `kubeconfig`

Generated secrets land in `infra/talos/secrets/`:

| File | Description |
|---|---|
| `infra/talos/secrets/talosconfig` | `talosctl` client config |
| `infra/talos/secrets/kubeconfig` | `kubectl` client config |

> **These files contain credentials.** They are listed in `.gitignore` and must not be committed.

---

## Step 5 — Bootstrap Flux

Install pre-Flux cluster components (CNI, optional cloud CCM) and bootstrap FluxCD from your git repository.

```shell
# From infra/ansible/
ansible-playbook \
  -i inventories/common \
  -i inventories/providers/<provider> \
  playbooks/bootstrap-flux.yml
```

This playbook:
1. Checks that `kubectl`, `helm`, and `flux` are available
2. Installs/upgrades Cilium (CNI) via Helm
3. Installs/upgrades the cloud CCM if `cloud_controller_manager.enabled` resolves to `true` for your provider
4. Runs `flux bootstrap` with the `repoUrl`, `repoBranch`, and `path` from your `cluster.yaml`
5. Pushes a deploy key to the repository and installs Flux controllers in `flux-system`

After this step, Flux owns ongoing cluster state.

---

## Post-Bootstrap: Accessing the Cluster

Use the generated credentials to access the cluster:

```shell
# kubectl
export KUBECONFIG=infra/talos/secrets/kubeconfig
kubectl get nodes

# talosctl
export TALOSCONFIG=infra/talos/secrets/talosconfig
talosctl -n <node-ip> health
```

---

## Post-Bootstrap: Verifying Flux

Check that Flux controllers are running and reconciling:

```shell
# See all Flux reconciliations and their status
flux get all

# Check Flux controller pods
kubectl -n flux-system get pods
```

What to expect: Flux watches `flux/clusters/<cluster-name>/` and reconciles the `infrastructure-controllers` and `infrastructure-configs` kustomizations defined under `flux/clusters/<cluster-name>/infrastructure/`. Additional cluster-specific infrastructure Kustomizations live in the same `infrastructure/` subdirectory, and per-app Kustomizations live under `flux/clusters/<cluster-name>/apps/`. On first bootstrap, Flux detects the pre-installed Cilium and CCM Helm releases and adopts them without reinstalling.

---

## Next Steps

- [Configure Your Cluster](configure-your-cluster.md) — change cluster settings, enable features, add Talos overlays
- [GitOps with Flux](gitops-with-flux.md) — day-2 operations, adding apps, monitoring Flux
- [Architecture Overview](../concepts/architecture-overview.md) — how the three subsystems fit together
