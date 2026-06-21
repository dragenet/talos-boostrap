# AGENTS.md

This repo is infrastructure-as-code for `kube1`, a Talos Linux Kubernetes cluster on Hetzner Cloud.

`CLAUDE.md` is the canonical source for architecture, workflows, repo status, and engineering standards. This file is the short OpenCode-facing entrypoint.

## Core Rules

- The top-level OpenCode session is an orchestrator. Delegate by default; do direct work only for trivial edits, tiny factual answers, or user-explicit no-subagent requests.
- For anything beyond trivial, use the orchestration skill, call `fast-plan` first unless durable planning is already clearly needed, and prefer subagents over broad work in the main session.
- Parallelize independent subagents when possible; preserve dependencies.
- Prefer multiple small, focused subagent tasks in parallel over one large bundled subagent task when the work can be split safely.
- Use graphify first for repo discovery. Use `repo-fast-context` only when graphify is insufficient or file/line evidence is needed.
- Use `web-fast-context` for official docs and external facts.
- Prefer graphify and the fast-context agents every time the main session needs repo context, not just at the start. Do not use `Read` or `Glob` in the main session until graphify and, when needed, `repo-fast-context` have narrowed the files for that specific question.
- Keep architectural decision-making in the live session or planners; `adr-writer` only writes ADR files for agreed decisions.
- Do not use the generic `explore` agent for kube1 repo questions.

## Subagents

- `fast-plan`: ephemeral read-only router for non-trivial but not deeply complex tasks.
- `task-planner`: durable implementation planner that writes `.ai/plans/`.
- `repo-fast-context`, `web-fast-context`: read-only context agents.
- `ansible-implementor`, `k8s-implementor`: implementation agents by domain.
- `ansible-reviewer`, `k8s-reviewer`: read-only reviewers.
- `docs-writer`, `adr-writer`, `report-writer`: docs/ADR/report specialists.

After implementation and successful review, write a report for multi-step work and then ask the user whether to commit. Use the global `git-commit` subagent only when the user explicitly says yes.

## Graphify

When `graphify-out/graph.json` exists, run `graphify query "<question>"` first for repo questions. Use `graphify explain` or `graphify path` for focused follow-up. Run graphify commands directly; do not wrap them in `echo`, pipes, or chained reminder commands.
