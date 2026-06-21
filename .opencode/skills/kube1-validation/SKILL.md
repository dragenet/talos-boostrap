---
name: kube1-validation
description: Use for validating kube1 Ansible, Talos, Flux, Kustomize, Helm, and documentation changes before handoff.
---

# kube1 Validation

Use this skill before handing changes back to the user.

## Ansible

- Run `ansible-lint` from `infra/ansible/` when Ansible files change.
- Use `ansible-playbook --check --diff` only with explicit approval when it may contact live infrastructure.
- Report exactly which inventory and playbook were validated.

## Flux and Kubernetes

- Run `kustomize build` or `flux build` for changed kustomization roots.
- Run `helm lint` or `helm template` for changed Helm releases/charts when feasible.
- Do not run live `kubectl` or `flux reconcile` without explicit approval.

## Docs and ADRs

- Check links/paths when practical.
- Keep implementation status honest: planned, stubbed, and running are different states.

## Reporting

Always report validations run, validations skipped, and why skipped validation was not safe or feasible.
