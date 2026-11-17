---
title: Split into tickets
description: Split an approved design into ordered tickets that each stand alone.
---

## What it does

The assistant reads an approved design document and splits its implementation plan into tickets. You choose how to split: by user journey, by layer, or by size. Each ticket lists the files it may change, the exact changes from the design, acceptance criteria, and the commands that prove it is done. Tickets are ordered so that the code builds after each one.

The assistant does not make design decisions while splitting. If a ticket needs something the design does not define, it asks you and records your answer as a design update.

A reviewer that has not seen the conversation then reads each ticket with only the design document. It reports tickets that need missing context, parts of the design that no ticket covers, and dependencies in the wrong order.

## When to use it

Use Split into tickets after Review, when the design is approved and the work is too large for one change or will be shared between people or agent sessions.

## What you get

A folder under `docs/tickets/<feature>/` with one Markdown file per ticket and a `README.md` index. The index has a dependency graph and a coverage table that shows which ticket covers each planned file, user journey, and verification step. Each ticket has frontmatter (`id`, `status`, `depends_on`, `covers`) that other tools can read.

## Example

```text
/egm-tickets docs/designs/webhooks.md
```

## Full instructions

The assistant follows the [Split into tickets skill instructions](../../../skills/egm-tickets/SKILL.md) in the repository.
