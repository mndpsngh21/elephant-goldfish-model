# EGM Ticket Breakdown (Approved Design to Actionable Tickets)

*   **Author:** Mandeep
*   **Status:** Proposed
*   **Date:** 2026-11-16

--------------------------------------------------------------------------------

## Overview

`/egm-tickets` reads an approved EGM design document and splits its Detailed
Implementation Plan into a set of small, ordered tickets. Each ticket is a
Markdown file that one engineer or one fresh agent session can pick up and
finish without reading the design conversation. A new Goldfish reviewer,
`egm-ticket-readiness`, checks the ticket set in a fresh context before the
files are written.

The skill sits after `/egm-review` in the feature flow:

```text
egm-brainstorm -> egm-draft -> egm-review -> egm-tickets -> implement -> egm-drift-report
```

## The Problem

EGM ends at an approved design document. The Detailed Implementation Plan in
that document lists every file to change, but it is one long list ordered by
file, not by unit of work. Teams then split it by hand, and three things go
wrong:

1.  **Lost coverage**: a file, a CUJ, or a verification step from the plan is
    not assigned to any ticket and is never built.
2.  **New decisions in tickets**: the person splitting the plan fills gaps
    (an interface between two tickets, an order of migrations) without the
    design author. The ticket then disagrees with the approved design.
3.  **Tickets that need the whole conversation**: a ticket says "implement the
    retry logic" and only someone who read the design discussion knows what
    that means. A fresh agent session given that ticket guesses.

The same Goldfish idea that EGM applies to designs applies here: a reviewer who
holds only one ticket and the design document finds the missing context.

## Goals

*   Turn an approved design document into tickets with no manual splitting.
*   Assign every planned file, CUJ, and verification item to at least one
    ticket, and show this in a coverage matrix.
*   Add no design decisions in tickets. A gap found while splitting goes back
    to the author as a question, or back to the design document.
*   Keep the codebase buildable and testable after each ticket, in dependency
    order.
*   Review the ticket set with a fresh-context Goldfish before writing it.
*   Write tracker-neutral Markdown files with YAML frontmatter, so other tools
    can import them later.

## Non-Goals

*   Creating issues in Jira, GitHub, or any other tracker. Import is a later
    step and is not part of this skill.
*   Estimating effort in hours or story points. Tickets carry only a rough
    size (S, M, L) derived from the number of files and changes.
*   Changing the design document. If splitting shows a gap, the skill stops and
    points the user to `/egm-draft` or `/egm-review`.
*   Implementing tickets or tracking their status after they are written.

## Open Questions

*   **Tracker import**: Should a later version add an optional export
    (Jira CSV, `gh issue create`) from the frontmatter? Out of scope here.

## Critical User Journeys (CUJs)

### CUJ 1: Split an approved design

*   **Given**: `docs/designs/webhooks.md` has `Status: Approved` and a Detailed
    Implementation Plan with eight target files.
*   **When**: The user runs `/egm-tickets docs/designs/webhooks.md` and picks a
    slicing strategy.
*   **Then**: The skill writes `tickets_draft.md`, runs the Goldfish review,
    and after approval writes `docs/tickets/webhooks/README.md` and one
    `T<NN>-<slug>.md` file per ticket. Every target file appears in the
    coverage matrix.

### CUJ 2: Splitting exposes a design gap

*   **Given**: Two planned files must share an interface that the design does
    not define.
*   **When**: The skill slices them into separate tickets.
*   **Then**: The skill does not invent the interface. It asks the author one
    question with up to three options, records the answer in the ticket, and
    tells the user to update the design document as well. If the author wants
    to update the design first, the skill stops.

### CUJ 3: Design is not approved

*   **Given**: The design document has `Status: Draft`.
*   **When**: The user runs `/egm-tickets`.
*   **Then**: The skill asks whether to run `/egm-review` first or continue,
    and records `design_status: Draft` in every ticket if the user continues.

## Technical Plan (Architecture)

```mermaid
sequenceDiagram
    autonumber
    actor Author
    participant Elephant as egm-tickets (main context)
    participant Loop as egm-review-loop (in-thread)
    participant Goldfish as egm-ticket-readiness + egm-codebase-details
    Author->>Elephant: /egm-tickets docs/designs/<feature>.md
    Elephant->>Elephant: Build plan inventory (F, C, V, G ids)
    Elephant->>Author: Slicing strategy gate (author decides)
    Elephant->>Elephant: Write tickets_draft.md + coverage self-check
    Elephant->>Loop: TargetFile=tickets_draft.md, GoldfishContext=[design doc, planned files]
    Loop->>Goldfish: Fresh-context review
    Goldfish-->>Loop: Findings
    Loop-->>Author: ticket_readiness_report.md + resolve gate
    Author->>Elephant: Approve
    Elephant->>Elephant: Write <TICKETS_DIR>/<feature>/README.md and T<NN>-<slug>.md
```

### Plan inventory

Before slicing, the Elephant reads the design and lists every item a ticket
must cover, with stable ids:

| Id | Source section | Item |
| :- | :------------- | :--- |
| `F<n>` | Detailed Implementation Plan, Target Files | One file entry, in plan order |
| `C<n>` | Critical User Journeys | One CUJ |
| `V<n>` | Verification Plan | One verification item |
| `G<n>` | Goals | One goal |

The inventory is the basis of the coverage matrix and the mechanical checks.

### Slicing strategy

How to split is an architectural choice with more than one defensible answer,
so the author picks it, following EGM's "Author Decides Architecture" rule:

*   **Vertical slices by CUJ**: each ticket delivers one user journey end to
    end. Fails when CUJs share most files.
*   **Layer by layer in plan order**: schema, then core logic, then API, then
    UI. Fails when early tickets cannot be tested without later ones.
*   **Size-capped increments**: about 200 to 400 changed lines per ticket,
    each leaving the build green. Fails when the plan has one large file.

### Ticket format

Each ticket has YAML frontmatter (`id`, `title`, `status`, `size`,
`depends_on`, `covers`, `design`, `design_status`) and these sections:
Summary, Context, Files, Changes, Acceptance Criteria, Verification, Out of
Scope. The full template is in
[`skills/egm-tickets/references/ticket.md`](../../skills/egm-tickets/references/ticket.md).

The Files section is a write allowlist: the implementer must not change any
file that is not listed there.

### Mechanical checks (Elephant, before review)

1.  Every `F`, `C`, and `V` id is covered by at least one ticket. A `G` id
    with no ticket is reported but does not block.
2.  Every planned file is owned by exactly one ticket, or is listed as shared
    with a reason in each ticket that touches it.
3.  No ticket lists a file that is not in the design's Target Files. Such a
    file is a design gap, not a ticket detail.
4.  `depends_on` has no cycles and no unknown ids. Tickets are numbered in a
    valid build order.

### Goldfish review

The skill reuses `egm-review-loop` in-thread, so convergence, resolution, and
report rules are the same as for designs:

*   `TargetFile`: `tickets_draft.md`, `full file`.
*   `Reviewers`: `egm-ticket-readiness`, `egm-codebase-details`.
*   `Standards`: both mapped to `ticket-readiness-standard`.
*   `GoldfishContext`: the design document and the planned files that exist.

The new persona reads the ticket set as an engineer who receives one ticket
and the design document, and nothing else. The new standard
([`ticket-readiness-standard.md`](../../resources/document_standards/ticket-readiness-standard.md))
defines the bar: self-contained, zero new decisions, complete coverage,
correct dependencies, buildable increments, testable acceptance criteria.

## Alternatives Considered

### Alternative A: Add a ticket section to the design document template

*   **Pros**: No new skill. The review already covers the design.
*   **Cons**: Mixes the "what" with the work split. The design document becomes
    stale as tickets change. The Stage 9 reviewer reads the plan, not per-ticket
    self-containment.
*   **Reason for Rejection**: Rejected by author: tickets have a different
    reader (one implementer, one ticket) and need their own review.

### Alternative B: Create tracker issues directly

*   **Pros**: One step from design to tracker.
*   **Cons**: Needs credentials and a connector per tracker. Hard to review and
    revise before creation.
*   **Reason for Rejection**: Rejected by author: Markdown keeps the skill
    tracker-neutral and reviewable. Frontmatter allows a later import step.

### Alternative C: Reuse `egm-implementation-readiness` as the reviewer

*   **Pros**: No new persona.
*   **Cons**: That persona checks one plan for missing decisions. It does not
    check coverage across tickets, dependency order, or whether one ticket is
    understandable without the others.
*   **Reason for Rejection**: Rejected by author: the new persona keeps the
    same zero-decision rule and adds the set-level checks.

--------------------------------------------------------------------------------

## Non-Functional Requirements

*   **Token cost**: one review round runs two reviewers. Convergence needs two
    clean rounds, capped at five, as in `egm-review-loop`.
*   **Safety**: the skill writes only `tickets_draft.md`, the review report,
    and files under the chosen tickets folder. It refuses to overwrite an
    existing ticket folder without the user's choice.

--------------------------------------------------------------------------------

## Detailed Implementation Plan

### Target Files to Create/Modify

#### 1. `skills/egm-tickets/SKILL.md` (Create)

*   **Type / Scope**: Skill definition.
*   **Proposed Changes**: Workflow steps 1 to 7: intake and approval check,
    plan inventory, slicing gate, draft, mechanical checks, Goldfish review
    through `egm-review-loop`, finalize.

#### 2. `skills/egm-tickets/references/ticket.md` (Create)

*   **Type / Scope**: Template.
*   **Proposed Changes**: Draft layout, ticket layout with frontmatter, and
    index (`README.md`) layout with coverage matrix and dependency graph.

#### 3. `skills/egm-tickets/{TEST.md,test_scenarios.yaml,agents/openai.yaml,assets/icon.svg}` (Create)

*   **Type / Scope**: Test plan, scenarios, Codex metadata, icon.
*   **Proposed Changes**: Follow the existing skill layout. Icon uses the
    drafting palette (`#1b6ef3`, `#7cacf8`).

#### 4. `agents/ticket_readiness.md` (Create)

*   **Type / Scope**: Canonical Goldfish persona.
*   **Proposed Changes**: Read-only reviewer with audit vectors for
    self-containment, zero-decision, coverage, dependencies, buildable
    increments, acceptance criteria, scope creep.

#### 5. `resources/document_standards/ticket-readiness-standard.md` (Create)

*   **Type / Scope**: Review standard.

#### 6. Host adapters (Create / Modify)

*   `.claude-plugin/agents/ticket_readiness.md` and its entry in
    `.claude-plugin/plugin.json`.
*   `.cursor-plugin/agents/ticket_readiness.md`.
*   `com.github.copilot/agents/egm-ticket-readiness.agent.md`.
*   `.windsurf/workflows/egm-tickets.md`.
*   `adapters/opencode/install.py`: add `ticket_readiness` to `REVIEWERS`.

#### 7. Packaging and tests (Modify)

*   `scripts/pack_plugin.py`: add the skill, persona, and standard to the
    required files.
*   `tests/unit/test_harnesses.py`, `tests/unit/test_packaging.py`,
    `tests/unit/test_windsurf.py`: register the new skill and reviewer.

#### 8. Documentation (Create / Modify)

*   `README.md`, `docs/getting_started.md`, `resources/README.md`,
    `docs/designs/README.md`, `docs/site/skills.md`,
    `docs/site/skills/egm-tickets.md`, and the site navigation.
*   `skills/egm-review/SKILL.md`: after Stage 9 concludes, mention
    `/egm-tickets` as the next step.

### Verification Plan

*   **Unit Tests**: `pytest tests/unit` passes.
*   **Lint**: `ruff check` passes.
*   **Manual**: run `/egm:egm-tickets` in Claude Code on an approved design
    with at least five target files; confirm the coverage matrix lists every
    `F` id and the Goldfish report is written.

--------------------------------------------------------------------------------

*Drafted with EGM*
