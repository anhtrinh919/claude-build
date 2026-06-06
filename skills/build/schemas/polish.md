# polish.md — Schema

> Agent context — not for human reading.
>
> Project-root file. Runs in parallel with `roadmap.md`. Use this to capture small bugs, polish requests, and minor friction the user raises **outside** the formal phase-spec process — things too small to be a new phase but too real to lose.

## Purpose

Roadmap = the planned, scope-defined work (phases). Polish = the unscheduled small stuff that surfaces during dogfood, between phases, or mid-session ("the button should be slightly bigger", "loading state stutters", "add an undo on the delete action"). Without a dedicated bucket these items either get forgotten or sneak into the current phase as scope creep.

## File location

`<project-root>/polish.md` — alongside `mission.md`, `roadmap.md`, etc.

## When agents touch this file

| Trigger | What happens |
|---|---|
| User says "log this for later" / "polish this later" / "add to the polish list" during any session | Append to the matching bucket below. One line per item. |
| `/review` finds a nit that is NOT a phase-blocker | Append to `## Open — Polish / minor` instead of forcing it into the current phase. |
| `/dogfood` (non-build pipeline) surfaces a small bug or wish | Append to `## Open — Bugs` or `## Open — Polish / minor`. |
| `/ba` Mode 2 (next-phase scoping) | **Read first.** Surface any open polish items that fit the next phase's scope — user can fold them in or defer. |
| Phase completes a polish item incidentally | Move the line from `## Open` to `## Done — Phase <N>` with a one-line note. |

## Template

```markdown
> Agent context — not for human reading.

# Polish & Backlog

Small bugs, polish, and unscheduled requests. Not phases (roadmap.md owns phases). Items here are unscheduled until folded into a phase by `/ba` Mode 2 or fixed incidentally.

---

## Open — Bugs
*(Defects in shipped behavior. One line each: <surface> — <what's broken> — <when reported>.)*

## Open — Polish / minor
*(UI tweaks, copy adjustments, micro-interactions. One line each: <surface> — <what to change> — <when reported>.)*

## Open — Requests
*(User asks for new tiny capabilities that don't justify their own phase. One line each: <what> — <why> — <when reported>.)*

## Open — Deferred from phase
*(Items pulled out of a phase spec late because they would have blown scope. Worth picking up later. One line each: <what> — <which phase deferred it>.)*

---

## Done

### Done — Phase 1
*(One line per item that was opened above and shipped, with a date.)*

### Done — Out of cycle
*(One-off fixes done without a phase. One line each.)*
```

## Rules

- One line per item. If something needs more than one line, it's a phase candidate, not a polish item — route it to `/ba` instead.
- Always include the date the item was reported (`YYYY-MM-DD`) — used by `/ba` Mode 2 to spot stale items.
- Do not delete done items — move them to `## Done` so the user can see what got addressed. Done items can be pruned manually if the file gets long.
- Do not use this file as a TODO for the current phase — current-phase work lives in `plan.md`. Polish is the **between-phase** bucket.
