# Pivot protocol

What counts as a pivot: a decision that reverses or materially changes an already-shipped phase's behavior, a settled `docs/decisions.md` entry, or the mission itself. This consolidates the stack's existing pivot mechanisms — it does not invent a new one.

**Mid-phase.** No special handling — it's a felt-impact fork like any other. Surface it via `AskUserQuestion` the moment it's recognized, in outcome language, with the tradeoff against the original plan named.

**After a phase ships.** Handled in `build-spec` replan mode: `CHANGELOG.md` gets an explicit `Pivoted:` / `Removed:` line under that phase's date heading (git's additive log doesn't show these). The `docs/decisions.md` entry it reverses is marked `superseded` — never silently deleted or overwritten; the ledger is append-only, so a later reader can see what changed and why.

**When it outgrows the mission.** `mission.md` is constitution-frozen — never silently edit it. If a phase's scope has genuinely expanded past what the product does or doesn't do, stop and surface it to the user explicitly as a **constitution change**: confirm the new mission line, then re-run only the affected slice of Step 1.4 (mission) or 1.5 (tech-stack constraints) — never the whole of Milestone 1 for a change this narrow. `CLAUDE.md`'s one-line product description refreshes only after `mission.md` was deliberately changed through this flow.
