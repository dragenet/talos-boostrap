---
name: kube1-orchestration
description: Use by default for kube1 tasks that are more complex than a trivial answer, tiny edit, or a few direct commands. Defines routing, subagent selection, context agents, parallelization, and when to escalate to task-planner.
---

# kube1 Orchestration

Use this skill by default for kube1 tasks that are not obviously trivial.

## Goal

Keep the main orchestrator small and reliable:

- Delegate by default.
- Use graphify first for repo discovery.
- Prefer graphify/LSP over broad work in the primary session; use web context only for external facts.
- Prefer opencode LSP tools for semantic navigation/refactors when available.
- Escalate to `task-planner` only when durable planning is clearly needed.
- Prefer multiple small, focused subagent tasks over one large bundled subagent task when the work can be split safely.
- For implementation, launch as many non-conflicting implementor agents in parallel as the plan safely allows. Keep each task small, simple, and scoped to one domain/context.

## Triggers

Use this skill when the task needs more than:

- brief reasoning,
- a few direct commands,
- or a direct answer to the user.

Do not use this skill for tiny factual answers or trivial one-file edits.

## Routing Rules

### 1. Trivial

Do it directly only when the task is clearly tiny and the subagent overhead adds no value.

### 2. Direct orchestration

For non-trivial work that does not need a durable plan, the main orchestrator should route directly using this skill:

- choose the lane,
- gather repo context via graphify/LSP,
- call `web-fast-context` for external facts,
- split implementation into small non-conflicting subagent tasks,
- preserve dependency order.

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
- Do not use `Read` or `Glob` in the primary session before graphify has narrowed the files for that specific question.
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
- `git`: git hygiene subagent, only when the user explicitly asks for git work or says yes to committing.

## Parallelization

Parallelize independent work whenever possible:

- independent graphify/LSP repo checks + `web-fast-context`
- independent Ansible and K8s implementors after planning
- independent reviewers after implementation

When parallelizing, prefer small focused subagent calls with clear scope over one oversized task that mixes multiple concerns.

For implementors specifically:

- Split work by non-overlapping files, domains, or contexts.
- Give each implementor a lightweight task with explicit scope and boundaries.
- Do not give one implementor a whole broad plan step if it contains multiple separable contexts.
- A good implementor task should fit this shape: gather local context, change one small scope, run static checks, report.
- If a task includes schema changes, renderer changes, role behavior, template migration, docs, and validation, split it before dispatch.
- Do not parallelize tasks that might edit the same file or depend on each other's output.
- If conflicts are possible, sequence those tasks instead.

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
- Do not use the generic `explore` agent for repo questions.
- Do not run broad raw shell repo searches in the primary session.
- Broad `bash: allow` is a convenience for local non-destructive work, not permission to take risky side effects silently.
- Ask the user before destructive, irreversible, live-cluster, provider-costing, secret-touching, or git history-rewriting commands.
