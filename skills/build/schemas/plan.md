# [Feature Name] Implementation Plan

Each group is independently reviewable and maps to one slice of the feature: a **sequence of work**, not a visual spec. The code-harness implements and verifies each group before the next.

## Ground rules for this file

- **Design-agnostic.** No hex values, no Tailwind classes, no pixel sizes, no font declarations. Those live in the design file (`design-tokens.css` + frame index in `handover.md`).
- **Behavior and sequence only.** State what each group delivers — capability, data flow, API surface, integration point — and which earlier groups it depends on.
- **Components are named, not described.** Name the component and its states; the design frame gives the visual half.
- **Each group has a verify line.** The command that proves it's done (e.g. `tsc -p . --noEmit`, `bun test --run CardDetail`, `verify-group-N.sh`). Groups build in dependency order; a group whose verify already passes is skipped (resume-idempotency).

## Group 1: [Category — e.g. "Shared primitives"]
**Delivers:** [one-line capability this group unlocks]
**Depends on:** [prior groups, or "none — scaffold"]
**Verify:** [command]

1. [Sub-task — named component / file / module, no visual detail]
2. [Sub-task]

## Group 2: [Category]
**Delivers:** ...
**Depends on:** Group 1
**Verify:** ...

1. [Sub-task]
2. [Sub-task]

## Group N: [Category]
...
