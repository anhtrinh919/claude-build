---
name: polish
description: Batch-mode backlog drainer, standalone and never auto-fired. Ingests open items from backlog.md plus an optional external source, diagnoses every ticket, groups them into high-impact-first batches, then works each item alone — diagnose, fix, verify, commit — pausing per batch for the user's dogfood.
user-invocable: true
argument-hint: "[optional filter — e.g. bugs, or an id range like B11-B16]"
---

# /build:polish — batched bug-fix mode

Standalone backlog drainer, a sibling to the pipeline rather than a step in it. `/build` never fires it; the user invokes it directly, between phases or after a dogfood pass. It drains **many** items in batches, one full loop per item and one dogfood pause per batch — distinct from `/build:review`'s in-pipeline gate and from a one-off scoped fix.

Every item ends `shipped-<commit>` or `blocked` with a recorded reason. No commit mixes two items. No `[symptom-patch]` or `[heuristic]` fix ships without recorded user opt-in.

Quoted lines are intent to convey, not scripts. IDs, statuses, paths, and the state schema are fixed.

## Invocation contract

| Mode | Model | Mechanism | Inputs | Outputs | Terminal state |
|---|---|---|---|---|---|
| single | Opus for orchestration, diagnosis, the hack gate, and commits; Sonnet for the ground-and-diagnose leaf, mechanical edits, and verify leaves | inline — owns commits and state writes (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md`) | `backlog.md`, an optional external source named at invocation, an optional filter arg | per-item commits, updated `backlog.md`, synced external items, per-batch and end reports | every item `shipped`/`blocked`/`deferred`, every batch `done`, `.polish-state.json` deleted on a clean drain |

## Discipline (non-negotiable)

1. Never hold two items' edits in one uncommitted tree.
2. **One item, one commit** — even inside a batch. A batch groups the work session and the pause point, never the commits.
3. Diagnose before patching, even when the fix looks like a one-liner. The obvious boundary fix is exactly where hacks hide.
4. A fix reaching into another item's area stops for a decision — fold it in deliberately and note it, or re-queue that item. Never let scope sprawl silently.
5. Never start a new batch before the user confirms they are done dogfooding the current one.

## Step 0 — Ingest and triage

**Always** read `backlog.md` — Reports (`DF-N`) and the flat buckets (`T-N`), schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/backlog.md`. Preserve IDs verbatim; never renumber.

**External source — ask every run, never assume or hardcode one.** One `AskUserQuestion`: just the backlog / the previously-used source (the default if `CLAUDE.md ## Project directives` names one) / other free text (a Notion page, a Linear, GitHub, or Jira view, any shared doc). Keep that source's own IDs verbatim (`B-n`, `ENG-n`, `#n`). A newly pasted source → offer to record it in `CLAUDE.md ## Project directives` as the next default.

Honor a filter arg if given (`/build:polish bugs`, `/build:polish B11-B16`).

Per open item capture: ID, a one-line symptom, evidence, **severity** (blocker / high-functional / medium / low-cosmetic), **subsystem**, and **dependencies** — does fixing it touch another item's area?

## Step 1 — Ground and diagnose every ticket

Before ordering anything, spawn one Sonnet leaf, context-isolated, with the full ticket list. It does the exploratory legwork for all tickets in one pass so the grouping below has real signal instead of guesses:

1. **Git check** per ticket — `git log --oneline -20` plus a grep for the ticket ID and symptom keywords. A recent commit already describing the fix → `auto-closed`, record the sha.
2. **File check** per ticket — a ticket naming a specific file, component, or route → confirm it still exists there. Renamed or removed → `stale`.
3. **Preliminary diagnosis** per surviving ticket — where the symptom likely manifests, a rough subsystem guess, and whether it reads as high-impact and visual or low-impact and cosmetic. This drives grouping only. It is **not** the authoritative root-cause call; Step 3 re-diagnoses on the main model before any fix.

Return one line per ticket: ID, verdict (`confirmed-open` / `auto-closed` / `stale`), subsystem guess, impact guess, one-line hypothesis. No commentary.

Apply the `auto-closed` and `stale` verdicts to `backlog.md` immediately and never queue them. Surface the prunes before proposing batches.

## Step 2 — Group into batches, then confirm

- **Batches group by shared code area or subsystem.** Closely-related bugs and improvements can mix — a batch is a work session, not a commit.
- **Order batches high-impact and visual first**: anything blocking or user-facing leads, functional-broken next, cosmetic last.
- **Within a batch**, sequence by dependency — fixing A precedes B if A touches B's area.

Surface the proposed batches (label, member IDs, a one-line reason each) via `AskUserQuestion` — confirm / reshuffle / reorder / drop. Do not start fixing until the plan is confirmed.

## Step 3 — Write the state file

`.polish-state.json` at the project root:

```json
{
  "batches": [
    { "label": "Checkout flow", "items": ["B16", "B12"], "status": "queued" }
  ],
  "queue": [
    { "id": "B16", "title": "…", "severity": "blocker", "batch": "Checkout flow", "status": "queued" }
  ],
  "currentBatch": 0,
  "currentIndex": 0,
  "filter": null
}
```

Item status ladder: `queued → diagnosing → fixing → verifying → shipped-<commit> | blocked | deferred`. Batch status ladder: `queued → active → awaiting-dogfood → done`.

**Read `.polish-state.json` first on any new turn** and resume at `currentBatch` and `currentIndex`. A batch found `awaiting-dogfood` means stop and re-ask the user before touching the next batch — never silently continue.

## Step 4 — Per-item loop

Repeat for the item at `currentIndex` within `currentBatch`.

1. **Restate** — ID, observable symptom, evidence. One plain line.
2. **Diagnose.** Locate where it manifests, then trace upstream to where the value or behavior is **first** wrong. Prefer the upstream fix over patching the boundary. Flag any fix that infers intent from surface form — a regex on formatting, a magic-string heuristic, a special case — as a smell. State the root cause in ONE line before writing code.
3. **Hack gate — a commit precondition.** Classify the fix. `[root-cause]` proceeds. `[symptom-patch]` or `[heuristic]` → surface the tradeoff (what the clean upstream fix would be, and why this is a band-aid) and proceed only on explicit user opt-in. Record the choice.
4. **Felt-impact fork.** A fix that forces a felt choice is a fork — surface it and apply the pick (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`). A bug with one correct outcome is not a fork; just fix it.
5. **Implement.** Root-cause reasoning stays on the main model; delegate mechanical edits to Sonnet or Haiku. Touch only this item's files.
6. **Verify** with `/build:review`'s browser engine and three-signal gate, scoped to this bug — its repro scenario plus a blind problem-resolution check — plus typecheck and the relevant tests. Not fixed until its specific symptom is gone on screen.
7. **Commit atomically and sync.** One ID, one commit; the body carries the one-line root cause and the classification tag. Update `backlog.md`. A named external source → tick its item, fetching first and editing in place, non-destructively. Set `shipped-<commit>` and advance `currentIndex`.

**Escalation.** An item still failing after **3 fix attempts** → `blocked` with a one-line reason. Leave it in place and **continue the batch**; a blocked item never halts the others.

## Step 5 — Batch pause

When `currentIndex` reaches the last item of the batch, shipped or blocked, **stop.** Do not roll into the next batch in the same turn.

1. Set the batch to `awaiting-dogfood`.
2. Report it: IDs shipped with their commits, IDs blocked with reasons.
3. Tell the user the batch is ready for their own hands-on dogfood, and ask whether to continue to the next batch, pause here, or re-open something in this one.
4. Only on "continue": set the batch `done`, advance `currentBatch`, reset `currentIndex`, and proceed.

## Step 6 — Wrap

Past the last batch, report by batch and then by ID: **shipped** with commits, **blocked** with reasons, **deferred or dropped**. Confirm `backlog.md` and any named external source are in sync. Delete `.polish-state.json` on a clean drain; leave it if any item is `blocked`, so a follow-up run resumes the unfinished set.

## Ground rules

1. **IDs are load-bearing.** State the ID every turn — start, fix, resolve. Never renumber.
2. **`backlog.md` is status truth; the external source is intake.** Tick the external source only when an item ships.
3. **The anti-hack gate is shared** with `/build:review`: a fix commit states where behavior is first wrong plus `[root-cause]`, `[symptom-patch]`, or `[heuristic]`. The last two need explicit user opt-in.
4. Brain auto-save fires per commit; a non-obvious root cause found during diagnosis qualifies.
