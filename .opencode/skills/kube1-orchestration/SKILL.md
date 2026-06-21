---
name: kube1-orchestration
description: Use by default for kube1 tasks that are more complex than a trivial answer, tiny edit, or a few direct commands. Defines routing, fast-plan usage, context agents, parallelization, and when to escalate to task-planner.
---

# kube1 Orchestration

Use this skill by default for kube1 tasks that are not obviously trivial.

## Goal

Keep the main orchestrator small and reliable:

- Delegate by default.
- Use graphify first for repo discovery.
- Prefer context subagents over broad work in the primary session.
- Prefer opencode LSP tools for semantic navigation/refactors when available.
- Prefer `fast-plan` for non-trivial routing by default.
- Escalate to `task-planner` only when durable planning is clearly needed.
- Prefer multiple small, focused subagent tasks over one large bundled subagent task when the work can be split safely.

## Triggers

Use this skill when the task needs more than:

- brief reasoning,
- a few direct commands,
- or a direct answer to the user.

Do not use this skill for tiny factual answers or trivial one-file edits.

## Routing Rules

### 1. Trivial

Do it directly only when the task is clearly tiny and the subagent overhead adds no value.

### 2. Fast-plan

Call `fast-plan` by default when the task is non-trivial and does not already clearly require `task-planner`:

- multiple likely next steps,
- uncertainty about lane/subagent choice,
- likely need for context gathering,
- likely need for validation planning,
- or likely benefit from parallelization.

`fast-plan` is ephemeral. It does not write `.ai/` artifacts itself, but it may escalate directly to `task-planner` when that boundary is clear. For most non-trivial tasks, the main orchestrator should call `fast-plan` before doing further discovery itself.

`fast-plan` should be a pure routing brain, not a discovery agent. It should reason only over the context already provided by the parent session. If more context is needed, it should tell the parent to gather it via graphify or the fast-context agents.

### 3. Task-planner

Call `task-planner` when the task is clearly bigger:

- multi-file or multi-domain implementation,
- architecture-affecting work,
- durable execution plan needed,
- long-running or dependency-heavy changes,
- or work that should be recorded in `.ai/plans/`.

## Context Rules

- For repo questions, run graphify first every time the main session needs new repo context.
- Prefer LSP-based symbol/definition/reference navigation when available for semantic code understanding or refactors.
- Prefer graphify and the fast-context agents every time repo context is needed; avoid broad manual file reads until the area is narrowed for that specific question.
- Do not use `Read` or `Glob` in the primary session before graphify and, when needed, `repo-fast-context` have narrowed the files for that specific question.
- Use `repo-fast-context` only when graphify is insufficient or file/line evidence is needed.
- Use `web-fast-context` for official docs, provider/tool facts, API details, release/version behavior, or external references.
- When repo and web context are independent, run both in parallel.
- Keep broad repo investigation out of the primary session whenever possible.

## Domain Routing

- `ansible-implementor`: Ansible, Talos patches, inventories, config rendering under `infra/`.
- `k8s-implementor`: Flux, Kubernetes, Helm, Kustomize under `flux/`.
- `ansible-reviewer`, `k8s-reviewer`: read-only review after implementation.
- `docs-writer`: operational docs and README/CLAUDE status updates.
- `adr-writer`: ADR file writing only after a decision is agreed.
- `report-writer`: final summary after multi-step work is complete.
- `git-commit`: commit hygiene subagent, only when the user explicitly asks to commit.

## Parallelization

Parallelize independent work whenever possible:

- `repo-fast-context` + `web-fast-context`
- independent Ansible and K8s implementors after planning
- independent reviewers after implementation

When parallelizing, prefer small focused subagent calls with clear scope over one oversized task that mixes multiple concerns.

Preserve dependencies:

- planning before implementation
- implementation before review
- report after review for multi-step work
- ADR only after decision agreement
- ask the user whether to commit after report handoff
- commit only if the user explicitly says yes

## Safety

- The user runs live cluster-changing commands.
- Read official docs before changing external tools/providers/controllers.
- Do not use the generic `explore` agent for kube1 repo questions.
- Do not run broad raw shell repo searches in the primary session.
