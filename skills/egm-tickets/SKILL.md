---
name: egm-tickets
license: Apache-2.0
disable-model-invocation: true
description: >-
  Splits an approved EGM design document into small, ordered, self-contained
  tickets written as Markdown files, then checks the ticket set with a
  fresh-context Goldfish reviewer. Use after egm-review approves a design and
  before implementation starts, when the Detailed Implementation Plan must be
  divided into units of work for engineers or agent sessions.
---

# Ticket Breakdown (Design to Actionable Tickets)

When running in Codex or ChatGPT Work, first read [Codex integration](../../docs/codex.md).
In Claude Code or Cowork, read `${CLAUDE_PLUGIN_ROOT}/docs/claude.md` first.

This skill turns the Detailed Implementation Plan of an approved design
document into tickets. A ticket is one unit of work that one engineer, or one
fresh agent session, can finish with only the ticket and the design document.
The skill splits and orders work. It does not design. Every decision in a
ticket must already be in the design document or be made by the author at a
gate in this skill.

## Delegation

A Goldfish runs in a fresh context with no conversation history. Spawn each
subagent with your harness's subagent tool (Antigravity: `invoke_subagent`; Claude Code: `Agent`; Codex: `spawn_agent`). Each subagent is defined in `agents/<file>.md`, and its
frontmatter `name` (for example `egm-ticket-readiness`) is the subagent type to
request where your harness registers named subagents. Otherwise spawn a
general-purpose subagent and tell it to read that persona file first. Pass the
prompt verbatim and wait for the report. If your harness cannot nest
subagents, use direct dispatch instead of the synthesizer. See
`docs/harnesses.md`. If your harness has no subagent tool, you may still write
the tickets, but tell the user that the Goldfish review in Step 6 did not run
and mark the index `Review: not run`.

## Execution Constraints (CRITICAL)

1.  You **MUST** read and apply the execution instructions in
    `resources/execution_instructions.md`. This skill writes tickets, not
    production code.
2.  **Artifact-First Delivery (MANDATORY)**: Do not paste the ticket set, the
    inventory, or review reports into chat. Write them with your file-writing
    tool and give the user the path.
3.  **Plain-English Standard**: Write tickets in simple, direct English. Name
    files, functions, fields, and commands exactly as the design names them.
    Do not use promotional adjectives or decorative adverbs.
4.  **No New Decisions (Zero-Decision Rule)**: Do not invent an interface, a
    field, an error case, an order of migrations, a dependency, or a file that
    the design does not define. When splitting needs such a decision, use the
    Design Gap Gate below.
5.  **No Glazing, No Apologies**: Do not praise the design or the user's
    choices. When corrected, state the corrected logic directly.
6.  **Interaction Invariants**: Every interactive gate (a structured question
    tool such as `ask_question`, if your harness has one, or plain chat) has
    exactly one question, is the only tool call in the turn, and gives
    action-oriented options without `(Recommended)` labels. Always write a
    short visible chat message before the gate.
7.  **Author Decides Architecture**: When a choice has more than one defensible
    answer, present up to three options, each with how it works, its
    trade-offs, and when it fails. Do not rank them or pick one. If the author
    says "just pick one", restate the trade-offs and show the same gate again.
    Decide mechanical choices yourself (ticket numbering, slugs, file names).

## Inputs

*   **`DesignFile`**: Path to the design document. If omitted, look for the most
    recently modified `docs/designs/*.md` whose `Status` is `Approved`, show its
    path, and ask the user in plain chat to confirm it or give another path.
    Ask for a path with a regular text message, not a structured question tool.
*   **`TicketsDir`** (Optional): Folder for the final tickets. If omitted, Step 7
    asks.

## Design Gap Gate

Use this gate whenever splitting needs a decision the design does not make.
Common cases: two tickets share an interface the design does not define, a
ticket cannot be tested without a stub the design does not mention, the plan
does not say which change must land first, or a needed file is missing from
Target Files.

1.  Write one line in chat that quotes the design text and names the gap.
2.  Present one single-select gate:

    ```json
    {
      "questions": [
        {
          "question": "Splitting needs a decision the design does not make: [gap]. How should we handle it?",
          "options": [
            "[Option A: mechanism; trade-off; fails when]",
            "[Option B: mechanism; trade-off; fails when]",
            "Stop and update the design document first"
          ],
          "is_multi_select": false
        }
      ]
    }
    ```

3.  If the author picks an option, record it in the affected tickets under
    **Context** as `Decided during ticket breakdown: [decision]`, and add it to
    the `Design Updates Needed` list in `tickets_draft.md`.
4.  If the author picks "Stop", list the gaps found so far in chat and tell the
    user to update the design with `egm-draft` or `egm-review` (use the host's
    command form, for example `/egm:egm-draft` in Claude Code). End the skill.

A file missing from Target Files is always a gap. Never add it on your own.

## Execution Workflow

Follow the steps in order. Do not skip a gate.

### Step 1: Intake and Approval Check

1.  Resolve `DesignFile` as described in Inputs. Read it in full.
2.  Confirm the design has a **Detailed Implementation Plan** with a **Target
    Files** list. If it does not, tell the user that tickets need that section
    and suggest running `egm-draft` Step 5. End the skill.
3.  Read the `Status` line. If it is not `Approved`, present one gate:
    *   Question: `"The design status is [Status], not Approved. How would you like to proceed?"`
    *   Option 0: `"Run egm-review on the design first"` → tell the user the
        host's command for `egm-review` with this path and end the skill.
    *   Option 1: `"Continue and mark every ticket with this design status"` →
        set `design_status` in every ticket to the current status.
4.  Read the **Open Questions** section. If it lists open questions, list them
    in chat and present one gate: `"Continue with open questions"` (each
    affected ticket names the question under Context) or `"Stop and resolve
    them in the design first"`.
5.  Read every file listed in Target Files that exists in the workspace, so the
    tickets use current names and paths.

### Step 2: Plan Inventory

Build the inventory and write it as the first section of `tickets_draft.md`
(layout in [references/ticket.md](references/ticket.md)). Use these ids, in the
order the design lists the items:

*   `F<n>`: each entry in Target Files. Keep the path and the action (Create,
    Modify, Delete).
*   `C<n>`: each Critical User Journey.
*   `V<n>`: each item in the Verification Plan.
*   `G<n>`: each Goal.

Do not merge, reword, or drop items. If the design groups several files in
one entry, keep one `F` id per file and note the group.

Notify the user in chat with the path and the counts (`F`, `C`, `V`, `G`).

### Step 3: Slicing Strategy Gate

Present one single-select gate. Fill each option with this design's facts (for
example, which CUJs share files, or which file is largest):

```json
{
  "questions": [
    {
      "question": "How should the plan be split into tickets?",
      "options": [
        "Vertical slices by CUJ: each ticket delivers one journey end to end; fails when CUJs share most files",
        "Layer by layer in plan order: data, core logic, interface, tests; fails when early layers cannot be tested alone",
        "Size-capped increments of about 200 to 400 changed lines, each leaving the build green; fails when one file is the whole change"
      ],
      "is_multi_select": false
    }
  ]
}
```

A write-in answer (for example, "one ticket per service") is valid. Restate it
in one line and use it.

### Step 4: Draft the Tickets

Write the tickets into `tickets_draft.md` in the working directory (or your
harness's artifact directory, if it has one), using the draft layout in
[references/ticket.md](references/ticket.md). Rules:

1.  **Order**: Number tickets `T01`, `T02`, ... in a valid build order. A
    ticket depends only on tickets with lower numbers.
2.  **Buildable increments**: After each ticket, in number order, the code
    builds and the tests listed so far pass. If a ticket can only pass with a
    later ticket, merge them or reorder them.
3.  **Files are an allowlist**: List every file the ticket creates, modifies,
    or deletes, with its `F` id. The implementer must not touch other files.
4.  **Changes come from the design**: For each file, state the changes the
    design specifies for this ticket, with names and signatures copied from the
    design. Link the design section, for example
    `[Detailed Implementation Plan, item 3](../designs/webhooks.md#3-...)`.
    Split one design entry across tickets only along a line the design already
    draws (for example, two functions it describes separately); otherwise use
    the Design Gap Gate.
5.  **Self-contained context**: The Context section says, in two to five
    sentences, why this ticket exists and what earlier tickets already
    provide. Do not refer to the chat ("as discussed").
6.  **Acceptance criteria**: Write Given/When/Then statements that a reviewer
    can check from the code or a test run. Reuse the design's CUJ wording when
    the ticket delivers a CUJ.
7.  **Verification**: Give the exact commands from the design's Verification
    Plan that apply to this ticket, and the test cases the ticket must add.
8.  **Size**: `S` (one or two files, no new public interface), `M` (three to
    five files, or one new interface), `L` (more). If a ticket is larger than
    `L` would suggest, split it again.
9.  **Out of scope**: Name what a reader might expect in this ticket but is in
    another ticket, with that ticket's id.

Then write the **Coverage Matrix** and **Dependency Graph** sections, and the
**Design Updates Needed** list (empty if no gap was decided in this skill).

### Step 5: Mechanical Checks

Check the draft yourself before any reviewer sees it. Fix failures in the
draft, or use the Design Gap Gate if a fix needs a decision.

1.  Every `F`, `C`, and `V` id appears in at least one ticket's `covers` list.
    A `G` id with no ticket is listed in chat but does not block.
2.  Every file is owned by exactly one ticket, or every ticket that touches it
    lists it as `shared` with a one-line reason.
3.  No ticket lists a file outside Target Files.
4.  Every `depends_on` id exists, points to a lower number, and the graph has
    no cycles.
5.  Every ticket has all sections from the template, with no `TBD`.

Write the result as a **Checks** list at the end of `tickets_draft.md` (one
line per check, `pass` or what failed and how it was fixed). Notify the user
in chat with the path, the number of tickets, and any check that needed a fix.
Then present one gate:

*   Question: `"Drafted [N] tickets in tickets_draft.md. How would you like to proceed?"`
*   Option 0: `"Run the Goldfish ticket review"`
*   Option 1: `"Request changes to the tickets (inline comments or write-in)"`
*   Option 2: `"Skip review and write the ticket files"`

On Option 1 with no write-in and no inline comments, ask in chat what to
change and end your turn. After changes, run Step 5 again. On Option 2, go to
Step 7 and mark the index `Review: skipped by author`.

### Step 6: Goldfish Ticket Review

Run `egm-review-loop` in this thread (read its `SKILL.md` and follow it; in
Claude Code do not use the `Skill` tool) with:

*   `TargetFile`: `tickets_draft.md` (absolute path)
*   `SemanticTarget`: `full file`
*   `Reviewers`: `[egm-ticket-readiness, egm-codebase-details]`
*   `Standards`: `{"egm-ticket-readiness": "ticket-readiness-standard",
    "egm-codebase-details": "ticket-readiness-standard"}`
*   `GoldfishContext`: the absolute path of `DesignFile`, plus every file from
    Target Files that exists in the workspace.

Pass these inputs directly; do not run the loop's intake questions again. The
loop writes its report and runs `egm-resolve`. Copy the final report to
`ticket_readiness_report.md`, tell the user its path, then present one gate:

*   Question: `"Ticket review completed. Report written to ticket_readiness_report.md. How would you like to proceed?"`
*   Option 0: `"Write the ticket files"`
*   Option 1: `"Resolve findings with agent"`
*   Option 2: `"I will update tickets_draft.md manually (notify me in chat when done)"`
*   Option 3: `"Pause (save progress and exit)"`

A finding that needs a design decision goes through the Design Gap Gate, not
`egm-resolve`. After any change to the draft, run Step 5 again before Step 7.

### Step 7: Write the Ticket Files

1.  **Target folder**: If `TicketsDir` was given, use it. Otherwise look for an
    existing `docs/tickets/` or `tickets/` folder, bind `[DiscoveredPath]`
    (default `docs/tickets/`), and present one gate: `"Save to detected
    directory: [DiscoveredPath]<feature>/"` or `"Specify custom target
    directory"`. `<feature>` is the design file name without `.md`.
2.  **No silent overwrite**: If `<TicketsDir>/<feature>/` already has files,
    present one gate: `"Write to a new folder <feature>-<YYYY-MM-DD>/"`,
    `"Replace the existing ticket files"`, or `"Cancel"`.
3.  **Write files**: Using the ticket and index layouts in
    [references/ticket.md](references/ticket.md), write one
    `T<NN>-<slug>.md` per ticket and a `README.md` index. Use relative links
    from the ticket folder to the design document. Set `status: todo` on every
    ticket.
4.  **Keep the draft**: Leave `tickets_draft.md` and the report in place.
5.  **Hand-off**: Tell the user the folder path, the number of tickets, the
    first ticket with no dependencies, and any item in `Design Updates
    Needed`. If that list is not empty, suggest updating the design with
    `egm-draft` so `egm-drift-report` does not report the tickets' decisions
    as drift later.
