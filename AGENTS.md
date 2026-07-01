# AGENTS.md

This repo is a reusable Talos Linux Kubernetes cluster deployment template.

`CLAUDE.md` is the canonical source for architecture, workflows, repo status, and engineering standards. This file is the short OpenCode-facing entrypoint.

## Core Rules

- The top-level OpenCode session is an orchestrator. Delegate by default; do direct work only for trivial edits, tiny factual answers, or user-explicit no-subagent requests.
- For anything beyond trivial, use the orchestration skill and prefer subagents over broad work in the main session. Use `task-planner` only when durable planning is clearly needed.
- Parallelize independent subagents when possible; preserve dependencies.
- Prefer multiple small, focused subagent tasks in parallel over one large bundled subagent task when the work can be split safely.
- For implementation, prefer launching as many non-conflicting implementor agents in parallel as the plan safely allows. Give each implementor a small, simple task scoped to one domain/context.
- Do not hand a whole plan step to one implementor if it contains multiple separable contexts. Split first; implementors should gather only local knowledge, implement one small scope, run static checks, and finish.
- Use graphify first for repo discovery. Use LSP and targeted reads only after graphify has narrowed the area.
- Use `web-fast-context` for official docs and external facts.
- Prefer graphify every time the main session needs repo context, not just at the start. Do not use `Read` or `Glob` in the main session until graphify has narrowed the files for that specific question.
- Keep architectural decision-making in the live session or planners; `adr-writer` only writes ADR files for agreed decisions.
- Do not use the generic `explore` agent for repo questions.
- Broad `bash` access means local non-destructive commands may run freely. Agents must ask before destructive, irreversible, live-cluster, provider-costing, secret-touching, or history-rewriting commands.

## Subagents

- `task-planner`: durable implementation planner that writes `.ai/plans/`.
- `web-fast-context`: read-only official-docs/external-facts context agent.
- `ansible-implementor`, `k8s-implementor`: implementation agents by domain.
- `ansible-reviewer`, `k8s-reviewer`: read-only reviewers.
- `docs-writer`, `adr-writer`, `report-writer`: docs/ADR/report specialists.

After implementation and successful review, write a report for multi-step work and then ask the user whether to commit. Use the global `git` subagent only when the user explicitly asks for git work or says yes to committing.

## Graphify

When `graphify-out/graph.json` exists, run `graphify query "<question>"` first for repo questions. Use `graphify explain` or `graphify path` for focused follow-up. Run graphify commands directly; do not wrap them in `echo`, pipes, or chained reminder commands.
