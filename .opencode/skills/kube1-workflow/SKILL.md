---
name: kube1-workflow
description: Use for kube1 repository work, especially plan -> implement -> review -> report flow, subagent routing, docs-before-code, and live-cluster safety.
---

# kube1 Workflow

Use this skill for non-trivial work in the kube1 repo.

## Flow

- Plan before implementing when work spans multiple files, layers, or domains.
- Route Ansible/Talos work to `ansible-implementor`; route Flux/Kubernetes work to `k8s-implementor`.
- Review with the matching read-only reviewer before handoff.
- After review passes, write operational docs when a user-visible workflow changes.
- After review passes, write a closing report for multi-step work under `.ai/reports/`.
- After report handoff, ask the user whether to commit the finished changes. Use `git` only if the user explicitly says yes or asks for git work.

## Safety

- The user runs live `ansible-playbook` commands against the cluster.
- Do not run provisioning, bootstrap, upgrade, `kubectl`, `flux reconcile`, `hcloud`, or `talosctl` commands without explicit approval.
- Flag cost-incurring or destructive operations before execution.
- Even when an agent has broad `bash` access, ask before destructive, irreversible, live-cluster, provider-costing, secret-touching, or git history-rewriting commands.
- Never expose or commit secrets, rendered Talos configs, decrypted SOPS files, or API tokens.

## Docs First

For external tools, libraries, plugins, controllers, and providers, fetch official docs or README before recommending or changing configuration.

For Hetzner server specs or pricing, fetch live Hetzner pages before quoting or configuring.
