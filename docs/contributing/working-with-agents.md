---
title: Working with Agents
description: How AI agents are orchestrated in this repo, the subagent roles, graphify usage, and what agents must not do without confirmation.
type: guide
audience: [contributor, ai]
tags: [agents, orchestration]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - docs-style-guide.md
---

# Working with Agents

This repo uses AI agents for implementation, documentation, review, and planning tasks. This guide describes how agents are orchestrated, what roles exist, and the constraints agents operate under.

---

## The orchestration contract

The canonical rules for agent orchestration are in [`AGENTS.md`](../../AGENTS.md) at the repo root. Read it before directing an agent to do work.

The key model:

- The **top-level session** is an orchestrator. It delegates by default and does direct work only for trivial edits or when the user explicitly asks.
- **Subagents** do the concrete work in parallel where tasks are independent.
- **Graphify** is the primary tool for repo discovery — agents run `graphify query` before reading files broadly.

---

## Subagent roles

| Role | Responsibility |
|---|---|
| `task-planner` | Writes durable implementation plans to `.ai/plans/` when multi-step work needs a recorded plan |
| `docs-writer` | Writes and updates files in `docs/` |
| `adr-writer` | Writes ADR files for decisions that have already been agreed in the live session |
| `ansible-implementor` | Implements changes in `infra/ansible/` |
| `k8s-implementor` | Implements changes in `flux/` and Kubernetes manifests |
| `ansible-reviewer` | Read-only review of Ansible work |
| `k8s-reviewer` | Read-only review of Kubernetes/Flux work |
| `report-writer` | Writes summary reports after multi-step implementation work |
| `web-fast-context` | Fetches official docs and external facts; read-only |

Agents should be given **small, focused tasks scoped to one domain**. Do not hand a whole plan step to a single implementor if it can be split safely.

---

## Graphify

When `graphify-out/graph.json` exists, agents use graphify for repo discovery before reading files directly.

```bash
# Ask a question about the codebase
graphify query "where are Talos machine config patches generated?"

# Explain a specific file or symbol
graphify explain infra/ansible/roles/config_render/

# Find the relationship between two files or concepts
graphify path config/clusters/ infra/talos/patches/generated/
```

Run graphify commands directly. Do not wrap them in `echo`, pipes, or chained commands.

If `graphify-out/graph.json` does not exist, fall back to targeted file reads and `Grep` searches after the orchestrator has narrowed the area.

---

## Plans and reports

- **Plans** live in `.ai/plans/`. They are historical artifacts — do not delete or modify them after the work begins.
- **Reports** live in `.ai/reports/`. The `report-writer` agent writes these after multi-step implementation work completes.

Both directories are informational. The repo's authoritative state is always the committed files, not the plans or reports.

---

## Writing good task descriptions for subagents

When dispatching a subagent, include:

1. **Scope** — exactly which files or directories the agent should read and write.
2. **Context** — the decision or requirement driving the work (link an ADR or plan if relevant).
3. **Constraints** — what the agent must not touch (e.g. "do not edit `config/defaults/`").
4. **Verification** — how the agent should confirm the work is correct (e.g. "run `pre-commit run ansible-lint`").
5. **Output** — what the agent should return (e.g. "return the list of files changed and any warnings").

Vague task descriptions produce vague results. The more precisely you define scope and constraints, the safer the subagent's actions.

---

## What agents must ask before doing

Agents run non-destructive commands freely. They **must ask for confirmation** before:

- Any command that is **destructive or irreversible** (deleting files, dropping databases, purging state).
- Any **live-cluster operation** (`talosctl apply-config`, `kubectl delete`, Flux reconciliation on a real cluster).
- Any **provider-costing operation** (creating cloud resources, resizing nodes).
- Any **secret-touching operation** (decrypting, re-encrypting, or rotating SOPS secrets).
- Any **history-rewriting git operation** (`git push --force`, `git rebase` on shared branches, amending pushed commits).

If an agent is unsure whether an operation falls into one of these categories, it should ask rather than proceed.
