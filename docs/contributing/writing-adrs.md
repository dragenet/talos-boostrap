---
title: Writing ADRs
description: How to write Architecture Decision Records for the template — when to write one, the format, and where to place it.
type: guide
audience: [contributor]
tags: [adr, contributing]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - docs-style-guide.md
  - ../adr/index.md
---

# Writing ADRs

Architecture Decision Records (ADRs) capture significant technical decisions alongside the context and reasoning that led to them. This guide explains when to write one, what format to use, and where to put it.

---

## When to write an ADR

Write an ADR when you are making a decision that:

- is difficult or costly to reverse (choice of OS, CNI, GitOps tool, storage driver);
- affects other contributors' mental model of the system;
- represents a meaningful trade-off between alternatives;
- would otherwise be re-litigated in future reviews.

**Do not** write an ADR for every small change. Routine config tweaks, dependency updates, and refactors that do not change the architecture do not need an ADR.

---

## ADR format

Existing ADRs in this repo follow a consistent format. Match it exactly.

### Frontmatter

```yaml
---
title: "ADR-NNN: Short title"
description: <one-line summary ≤ 140 chars>
type: adr
audience: [contributor, ai]
tags: [adr, <topic>, ...]
status: accepted
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

Use `status: accepted` for current decisions. Use `status: superseded` when a later ADR replaces this one, and add a cross-reference in the body.

### Body structure

```markdown
# ADR-NNN: Short title

**Status:** Accepted

## Context

<Why does this decision need to be made? What forces are in play?>

## Decision

<What was decided? Be precise about the choice and how it will be implemented.>

## Consequences

<What does this mean going forward? List trade-offs, constraints, and effects on the rest of the system.>
```

The three mandatory sections are **Context**, **Decision**, and **Consequences**. Keep each section focused:

| Section | Answers |
|---|---|
| Context | Why is a decision needed? What constraints exist? |
| Decision | What did we choose, and how does it work? |
| Consequences | What are the trade-offs and downstream effects? |

You may add an **Addendum** section at the end to record later clarifications or corrections without altering the original decision. See ADR-011 for an example.

---

## Where to put it

| Scope | Location |
|---|---|
| Generic template decisions (apply to all users) | `docs/adr/` |
| Provider-specific decisions for a worked example | `docs/examples/<provider>/adrs/` |

Most ADRs belong in `docs/adr/`. Provider-specific placement is only appropriate for decisions that are meaningful only within that concrete deployment.

---

## Numbering

Check the highest-numbered ADR in the target directory and use the next available number, zero-padded to four digits:

```bash
ls docs/adr/ | grep -E '^[0-9]{4}' | sort | tail -1
```

Name the file `<NNNN>-short-kebab-title.md`.

---

## Checklist before committing

- [ ] Frontmatter is complete and follows the schema above
- [ ] `status` is `accepted`
- [ ] The body has all three sections (Context, Decision, Consequences)
- [ ] `description` is ≤ 140 characters
- [ ] The file is in the correct directory
- [ ] The file name matches the ADR number
- [ ] The `docs/adr/index.md` table includes a row for the new ADR
