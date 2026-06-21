---
name: kube1-architecture
description: Use when kube1 work may introduce or change architectural decisions, ADR requirements, topology, provider seams, storage, networking, GitOps, or security posture.
---

# kube1 Architecture Decisions

Use this skill when a task touches topology, provider seams, Talos/Kubernetes bootstrap boundaries, storage, networking, ingress, security, GitOps ownership, or operational blast radius.

## Decision Ownership

- Architectural planning happens in the live session or `task-planner`.
- Implementors may discover decision points but must not silently decide durable architecture in code.
- `adr-writer` writes ADR files only after the decision is agreed.

## ADR Trigger

Create or update an ADR when a change introduces or changes:

- A topology or provider choice.
- A bootstrap or GitOps boundary.
- A storage/networking/security pattern.
- A tool/controller/pattern with meaningful trade-offs or rejected alternatives.
- A durable operational contract other contributors must understand later.

## ADR Location

- Provider-agnostic bootstrap decisions: `docs/cluster-bootstrap/adrs/`.
- kube1-specific or provider-specific decisions: `docs/kube1/adrs/`.

## Required Content

Capture context, decision, consequences, and rejected alternatives. Tie trade-offs to HA, latency, cost, security, and blast radius where relevant.
