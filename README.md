# kube1

Infrastructure-as-code for a 3-node Talos Linux Kubernetes cluster on Hetzner Cloud.

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
