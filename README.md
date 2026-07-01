# Talos Kubernetes Bootstrap Template

Infrastructure-as-code template for deploying Talos Linux Kubernetes clusters.
The current implementation includes Ansible provisioning/bootstrap flows, Talos
configuration rendering, and Flux GitOps manifests.

For architecture decisions, engineering standards, workflows, and the current
implementation status, see [docs/index.md](./docs/index.md) — it is the canonical
entrypoint for both humans and AI agents working on this repo.

## Quick start

See [docs/index.md](./docs/index.md) → "Getting Started" and the guides section for
the full bootstrap sequence (render → provision → bootstrap Talos → bootstrap Flux).

## Repository layout

- `infra/ansible/` — VM provisioning and cluster bootstrap (Ansible)
- `infra/talos/` — Talos machine config patches and generated secrets (gitignored)
- `flux/` — FluxCD GitOps manifests
- `docs/` — the documentation vault (guides, concepts, reference, ADRs, contributing)
