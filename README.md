# Talos Kubernetes Bootstrap Template

Infrastructure-as-code template for deploying Talos Linux Kubernetes clusters.
The current implementation includes Ansible provisioning/bootstrap flows, Talos
configuration rendering, and Flux GitOps manifests.

For architecture decisions, engineering standards, workflows, and the current
implementation status, see [CLAUDE.md](./CLAUDE.md) — it is the canonical source of
truth for both humans and AI agents working on this repo.

## Quick start

See `CLAUDE.md` → "Prerequisites" and "Key workflows" for the full bootstrap sequence
(image build → provision → bootstrap → Flux).

## Repository layout

- `infra/ansible/` — VM provisioning and cluster bootstrap (Ansible)
- `infra/talos/` — Talos machine config patches and generated secrets (gitignored)
- `flux/` — FluxCD GitOps manifests
- `docs/` — architecture decisions and runbooks
