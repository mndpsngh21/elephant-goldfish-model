# Ticket Readiness Standard

This standard defines the quality bar for a set of tickets produced from an
approved design document.

## Core Principle

A ticket passes if an engineer (or a fresh AI session) who receives only that
ticket and the design document it links can make the correct change on the
first attempt, without asking questions and without making design choices.
The ticket set passes if, done in dependency order, it builds everything the
design's Detailed Implementation Plan describes, and nothing else.

## The Zero-Decision Rule

Tickets add no design decisions. Every interface, field, error case, file, and
ordering constraint in a ticket must come from the design document, or be
listed in the draft's **Design Updates Needed** section as a decision the
author made during the breakdown. A ticket that leaves such a choice to the
implementer fails.

## Requirements

### Per ticket

*   **Self-contained**: The Context section explains why the ticket exists and
    what earlier tickets provide, and links the design sections it implements.
    It does not rely on conversation history or on text found only in other
    tickets.
*   **File allowlist**: The Files table lists every file the ticket creates,
    modifies, or deletes, by exact repository-relative path, with its inventory
    id. Every listed file appears in the design's Target Files.
*   **Exact changes**: For each file, the Changes section names the functions,
    classes, fields, signatures, and error handling as the design states them.
    "Update X to support Y" without these details fails.
*   **Checkable acceptance**: Each acceptance criterion is a Given/When/Then
    statement that can be checked from the code or a test run.
*   **Verification**: At least one exact command, and the test cases the ticket
    must add.
*   **Out of scope**: Names related work that belongs to another ticket, with
    that ticket's id.

### Across the set

*   **Complete coverage**: Every Target File, CUJ, and Verification item in the
    design is in the Plan Inventory and is covered by at least one ticket.
*   **Single ownership**: Each file has one owning ticket. Other tickets that
    touch it mark it `shared` with a reason, and their changes do not conflict.
*   **Valid order**: Tickets are numbered in build order. A ticket depends only
    on lower-numbered tickets. The dependency graph has no cycles.
*   **Buildable increments**: After each ticket and its dependencies, the code
    builds and the ticket's verification passes. No ticket needs code from a
    later ticket.
*   **Consistent tables**: The Coverage Matrix and Dependency Graph match the
    tickets' `Covers` and `Depends on` fields.
