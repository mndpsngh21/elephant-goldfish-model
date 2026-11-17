---
description: Route to canonical skills/egm-tickets/SKILL.md without copied skill text.
---

1. Read `skills/egm-tickets/SKILL.md` from the EGM package root.
2. Execute the skill.
3. Resolve `agents/*.md` and `resources/document_standards/*.md` relative to the package root.
4. Preserve all required human approval gates and output boundaries from `skills/egm-tickets/SKILL.md`.
5. Run the Step 6 Goldfish review under the `egm-review-loop` isolation rules in `.windsurf/rules/egm.md`.
