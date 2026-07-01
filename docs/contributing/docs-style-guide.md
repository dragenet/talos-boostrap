---
title: Docs Style Guide
description: The authoring contract for all docs/ vault notes — frontmatter schema, link style, section taxonomy, note types, and prose rules.
type: guide
audience: [contributor, ai]
tags: [contributing, docs, style]
status: stable
created: 2026-07-01
updated: 2026-07-01
related:
  - index.md
  - ../index.md
---

# Docs Style Guide

This guide defines the authoring contract for every file in `docs/`. Follow it when writing new notes, migrating existing content, or reviewing contributions.

---

## 1. Frontmatter schema

Every `.md` file in `docs/` **must** begin with a YAML frontmatter block:

```yaml
---
title: <human title>
description: <one-line summary for humans and AI retrieval>
type: guide            # guide | concept | reference | adr | index
audience: [user]       # any of: user | contributor | operator | ai
tags: [config, talos]  # topical tags
status: stable         # stable | draft | accepted | superseded
created: 2026-07-01
updated: 2026-07-01
related:               # relative paths as plain strings (optional)
  - ../concepts/config-compiler.md
---
```

### Field rules

| Field | Rules |
|---|---|
| `title` | Human-readable heading; match the H1 in the body. |
| `description` | ≤ 140 characters. This is the AI retrieval snippet — make it precise. |
| `type` | One of `guide`, `concept`, `reference`, `adr`, `index`. See §4. |
| `audience` | Array of applicable roles. Use `ai` when the note is primarily consumed by agents. |
| `tags` | Topical keywords in kebab-case. |
| `status` | `stable` — reviewed and current. `draft` — work in progress. `accepted` / `superseded` — for ADRs only. |
| `created` / `updated` | ISO date (`YYYY-MM-DD`). Update `updated` on every meaningful edit. |
| `related` | Optional list of relative paths to closely related notes. Plain strings, no YAML anchors. |

Use `type: index` for MOC/hub notes (`index.md` in each section folder).  
Use `type: adr` and `status: accepted` or `status: superseded` for all ADR files.

---

## 2. Link style

Use **standard relative markdown links** everywhere:

```markdown
[Config Compiler](../concepts/config-compiler.md)
[Getting Started](../guides/getting-started.md)
```

**Do not** use `[[wikilinks]]` in body prose. Wikilinks are Obsidian-only and break on GitHub and most tools.

The `related` frontmatter field lists note paths as plain strings (not markdown links):

```yaml
related:
  - ../concepts/config-compiler.md
  - ../adr/0011-config-compiler.md
```

When linking across sections, always use paths relative to the current file. Verify that every link resolves to an existing file before committing.

---

## 3. Section taxonomy

| Section | What belongs here |
|---|---|
| `guides/` | Task-oriented how-to guides. Answer "how do I do X?" Steps are numbered. Imperative headings (e.g. "Install prerequisites"). |
| `concepts/` | Explanation and understanding. Answer "what is X and why does it work this way?" Noun-phrase headings. No step-by-step tasks. |
| `reference/` | Lookup tables, schemas, inventories, and glossaries. Answer "what are the exact values/options for X?" Tabular where possible. |
| `adr/` | Architecture decision records for generic template decisions. See [Writing ADRs](writing-adrs.md). |
| `contributing/` | Contributor workflow: dev setup, docs authoring, ADR process, CI, and working with AI agents. |
| `examples/` | Concrete provider/instance worked examples. Each sub-folder is a self-contained reference for one real deployment. |

If content fits multiple sections, prefer the most specific one. A guide that requires conceptual background should link out to `concepts/` rather than embedding the explanation inline.

---

## 4. Note types

### `guide`

A step-by-step task walkthrough. Has a clear goal, numbered steps, and expected outcomes. Written in second-person, imperative voice. Examples: `getting-started.md`, `add-a-provider.md`.

### `concept`

An explanation of a system, model, or decision. Builds understanding rather than prescribing actions. No numbered steps. Examples: `architecture-overview.md`, `config-compiler.md`.

### `reference`

A lookup document. Contains tables, schemas, or enumerated options. Minimal prose. Examples: `cluster-yaml-schema.md`, `glossary.md`, `playbooks.md`.

### `adr`

An Architecture Decision Record. Records a decision, its context, considered alternatives, and consequences. Uses the standard ADR template (`status`, `date`, `context`, `decision`, `consequences`). See [Writing ADRs](writing-adrs.md).

### `index`

A Map of Content (MOC) hub note. Lists pages in a section with one-line descriptions and relative links. Every section folder has exactly one `index.md` with `type: index`.

---

## 5. Voice — template-first, provider-agnostic

Primary docs in `guides/`, `concepts/`, and `reference/` are **template-voice**:

- Never assume a specific cloud provider (Hetzner, AWS, etc.).
- Never hardcode a cluster name (`kube1`, `prod`, etc.).
- Use generic placeholders: `<cluster-name>`, `<provider>`, `your-cluster`.

Provider- and instance-specific material belongs exclusively under `examples/<provider-instance>/`. Notes there are explicitly labeled as example instances and may mention Hetzner, specific IPs, node names, etc.

---

## 6. Prose style

- **Short paragraphs.** One idea per paragraph. Aim for 2–4 sentences.
- **Active voice.** Prefer "Run the playbook" over "The playbook should be run."
- **Second person.** Address the reader as "you." ("You will need…", "Run the following…").
- **Imperative headings for guides.** Use verb phrases: "Install prerequisites", "Bootstrap the cluster", "Rotate certificates".
- **Noun-phrase headings for concepts and reference.** Use nouns: "Config compiler model", "Layered configuration", "Inventory contract".
- **Code blocks.** Always specify a language for syntax highlighting: ` ```yaml `, ` ```bash `, ` ```text `.
- **Admonitions (optional).** Use `> **Note:**`, `> **Warning:**`, `> **Tip:**` callouts sparingly, only when a detail would otherwise be missed.
- **Avoid future tense.** Write as if describing current behavior, not planned behavior. If something is not yet implemented, mark the note `status: draft`.
