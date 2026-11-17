# Test Plan: EGM Tickets Skill

Frontmatter and link checks run in `pytest tests/unit` (see [TESTING.md](../../TESTING.md)).

## Protocol Verification Checklist

When evaluating `egm-tickets` interactively in chat, verify:

-   **Step 1 (Intake):** Stops when the design has no Detailed Implementation
    Plan. Asks before continuing with a design that is not `Approved`.
-   **Step 2 (Inventory):** Lists one `F` id per target file, and every CUJ,
    Verification item, and Goal, without merging or dropping any.
-   **Step 3 (Slicing):** Asks the author to choose; does not pick a strategy
    when told to.
-   **Design Gap Gate:** Does not add a file, interface, or field that the
    design lacks. Records each author decision under Design Updates Needed.
-   **Step 5 (Checks):** Writes the Checks list and fixes coverage, ownership,
    and dependency errors before review.
-   **Step 6 (Review):** Dispatches `egm-ticket-readiness` and
    `egm-codebase-details` in fresh contexts with the design document in
    `GoldfishContext`.
-   **Step 7 (Write):** Does not overwrite an existing ticket folder without a
    choice. Writes one `T<NN>-<slug>.md` per ticket and a `README.md` index.
