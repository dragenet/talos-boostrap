---
title: Development Setup
description: How to set up a local development environment for contributing to the Talos Kubernetes bootstrap template.
type: guide
audience: [contributor]
tags: [setup, development]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - testing-and-precommit.md
  - docs-style-guide.md
---

# Development Setup

This guide walks you through setting up a local environment for working on the template. Complete all steps before making your first change.

---

## Install prerequisites

You need the following tools installed and available on your `PATH`.

### Talos

| Tool | Purpose |
|---|---|
| [`talosctl`](https://www.talos.dev/latest/introduction/getting-started/) | Interact with Talos nodes; generate and apply machine configs |

### Kubernetes

| Tool | Purpose |
|---|---|
| [`kubectl`](https://kubernetes.io/docs/tasks/tools/) | Kubernetes CLI |
| [`helm`](https://helm.sh/docs/intro/install/) | Helm chart management |
| [`flux`](https://fluxcd.io/flux/installation/) | FluxCD GitOps CLI |

### Ansible

| Tool | Purpose |
|---|---|
| `ansible` | Run playbooks (bootstrap, render-config, etc.) |
| `ansible-lint` | Enforced by the `ansible-lint` pre-commit hook |

Install Ansible via your package manager or `pip`. The pre-commit hook uses the version pinned in `.pre-commit-config.yaml` (`v26.4.0`).

### Secrets

| Tool | Purpose |
|---|---|
| [`sops`](https://github.com/getsops/sops) | Encrypt and decrypt secrets |
| [`age`](https://github.com/FiloSottile/age) | Key backend for SOPS (see `.sops.yaml`) |

SOPS is configured to encrypt files matching `infra/ansible/inventories/*/group_vars/*.sops.yaml` or `infra/ansible/inventories/*/group_vars/*.sops.yml` using an `age` public key. You will need your own `age` key pair to read and write encrypted secrets.

### Local tools

| Tool | Purpose |
|---|---|
| [`pre-commit`](https://pre-commit.com/) | Run all linting hooks locally before push |

---

## Set up pre-commit hooks

Install the hooks so they run automatically on every `git commit`:

```bash
pre-commit install
```

To verify the installation, run all hooks against the full repo:

```bash
pre-commit run --all-files
```

All hooks should pass on a clean checkout. If any fail, fix the reported issues before proceeding. See [Testing and Pre-commit](testing-and-precommit.md) for details on each hook.

---

## Set up SOPS age keys

The repo uses `age` for secret encryption. SOPS will encrypt to the public key configured in `.sops.yaml`. To decrypt existing secrets, you need the corresponding private key.

1. If you have an existing `age` key pair, locate the private key file (typically `~/.config/sops/age/keys.txt` or the path set in `$SOPS_AGE_KEY_FILE`).

2. If you are creating a new key pair for a fresh environment:

   ```bash
   age-keygen -o ~/.config/sops/age/keys.txt
   ```

   Share your new **public key** with the project maintainer. The maintainer must add it to `.sops.yaml` and re-encrypt all secret files before you can decrypt them.

3. Point SOPS at your private key file:

   ```bash
   export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
   ```

   Add this export to your shell profile to persist it.

4. Verify access by decrypting a secrets file:

   ```bash
   sops --decrypt infra/ansible/inventories/<provider>/group_vars/<provider>.sops.yaml
   ```

> **Note:** Never commit unencrypted secrets. All files matching the path regex in `.sops.yaml` must be encrypted with `sops` before committing.

---

## Validate your setup

Run the full pre-commit suite to confirm everything is installed correctly:

```bash
pre-commit run --all-files
```

If `ansible-lint` passes, your Ansible installation is compatible with the pinned hook version. If it fails with a version error, check that `ansible` and `ansible-lint` are installed and on your `PATH`.
