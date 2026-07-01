---
title: "Example: kube1 on Hetzner Cloud"
description: Worked example of the template deployed as a real cluster (kube1) on Hetzner Cloud — instance-specific config, ADRs, and operational notes.
type: index
audience: [user, operator]
tags: [example, hetzner, kube1]
status: draft
created: 2026-07-01
updated: 2026-07-01
related:
  - ../../index.md
  - adrs/index.md
---

# Example: kube1 on Hetzner Cloud

> **This is an example instance, not part of the generic template.** Content here documents a specific deployment of the template and may reference Hetzner-specific resources, node names, and configuration values.

`kube1` is a production Kubernetes cluster deployed on [Hetzner Cloud](https://www.hetzner.com/cloud) using this template. It serves as a worked reference for how to adapt the template to a real provider.

## Contents

| Page | Description |
|---|---|
| [ADRs](adrs/) | Instance-specific architecture decision records for kube1 (ADRs 001–005) |

_Additional example content (config snippets, provider-specific notes) will be added in later steps._

## Relationship to the template

The generic template docs live in the parent vault sections (`guides/`, `concepts/`, `reference/`). This example shows how those generic patterns are applied in practice. Where this example deviates from template defaults, the deviation is noted and explained.
