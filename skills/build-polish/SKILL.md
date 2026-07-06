---
name: build-polish
description: Batch-mode backlog drainer for the /build stack — standalone, never auto-fired by the orchestrator. Ingests open items from backlog.md (T-N / DF-N) plus an optional external source named at invocation (Notion / Linear / GitHub / a shared doc — never assumed or hardcoded), spawns a Sonnet subagent to ground + diagnose every ticket up front, then groups closely-related bugs/improvements into batches (high-impact/visual first), proposes the batches to the user. Works each batch's items in isolation — restate → root-cause diagnose → anti-hack gate → fix → scoped verify (via /build-review's browser engine) → atomic commit → sync backlog + external source — then pauses after the batch for the user's own hands-on dogfood before continuing. One item is still one commit; batching groups the queue and the pause point, never the commits. Resumable via .polish-state.json. Trigger on /build-polish, "work the backlog", "fix the open bugs by batches", "polish pass", "drain the tracker".
---

# /build-polish — Bug-Fix Mode (batched, with discipline)

Standalone backlog-drainer for the `/build` stack — a sibling, not a pipeline step. `/build` never fires this; the user invokes it directly, between phases or after a dogfood pass. Distinct from `/build-review` (in-pipeline phase gate) and a one-off scoped fix (no skill needed): this drains **many** items, grouped into batches, one full loop per item and one dogfood pause per batch.

## What this is

- **Goal.** Close open bugs/requests in high-impact-first batches, each item with a named root cause, an honest fix, scoped verification, atomic commit — backlog and any named external source end in sync, with the user hands-on-dogfooding between batches instead of waiting for the whole drain.
- **How.** Ingest `backlog.md` + optional named external source → Sonnet subagent grounds + diagnoses every ticket in one pass → group into batches (closely-related bugs/improvements can mix; high-impact/visual-first order), user-confirmed → per item in the active batch: diagnose → anti-hack gate → mechanical edit (Sonnet/Haiku) → verify via `/build-review`'s three-signal gate scoped to this bug → atomic commit → sync → after the batch's last item, pause for the user's own dogfood before starting the next batch. `.polish-state.json` survives compaction.
- **Success.** Every queued item ends `shipped-<commit>` or `blocked` (reason recorded); no commit mixes two items; no `[symptom-patch]`/`[heuristic]` fix ships without recorded user opt-in.

**Discipline, non-negotiable:** never hold two items' edits in one uncommitted tree · one bug = one commit, even inside a batch — grouping items into a batch never means merging their fixes · diagnose before patching, even when the fix looks like a one-liner (the obvious boundary fix is exactly where hacks hide) · a fix reaching into another item's area stops for a decision — fold deliberately and note it, or re-queue that item — never silent scope sprawl · never start a new batch before the user has confirmed they're done dogfooding the current one.

A quoted line below is intent to convey, not a script to recite. Phrasing to the user is yours; ids, statuses, file paths, and state schema are fixed.

## Invocation contract

| Model | Mechanism | Inputs | Outputs | Terminal state |
|---|---|---|---|---|
| Opus: orchestration, diagnose, hack-gate, commits. Sonnet: ground+diagnose subagent, mechanical edits, verify subagents. | inline (owns commits + state writes; one-level nesting per `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md`) | `backlog.md`, optional external source named at invocation, optional filter arg | per-bug commits, updated `backlog.md`, synced external-source items, per-batch + end report | every item `shipped`/`blocked`/`deferred`; every batch `done`; `.polish-state.json` deleted on clean drain |

## Step 0 — Ingest + triage (don't fix yet)

**Always:** `backlog.md` — Reports (`DF-N`) + flat buckets (`T-N`: roll-in / dogfood-polish / side), schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/backlog.md`. Preserve ids verbatim, never renumber.

**External source — ask every run, never assume or hardcode one.** One `AskUserQuestion`: just the backlog / the previously-used source (default if `CLAUDE.md ## Project directives` names one) / other free-text (a Notion page, a Linear/GitHub/Jira view, any shared doc). Keep the source's own ids verbatim (`B-n`, `ENG-n`, `#n`, …). New source pasted → offer to record it in `CLAUDE.md ## Project directives` as the next default.

Honor a filter arg if given (`/build-polish bugs`, `/build-polish B11-B16`).

Per open item capture: id, one-line symptom, evidence, **severity** (blocker / high-functional / medium / low-cosmetic), **subsystem**, **dependencies** (does fixing it touch another item's area?).

## Step 0b — Ground + diagnose every ticket (Sonnet subagent)

Before ordering anything, spawn one Sonnet subagent (context-isolated inline brief, per `subagent-policy.md` Rule 2/3) with the full ingested ticket list. It does the exploratory legwork for all tickets in one pass, so grouping/ordering in Step 1 has real signal instead of guesses:

1. **Git check** per ticket — `git log --oneline -20` + grep the ticket id/symptom keywords. A recent commit already describing the fix → `auto-closed` (record the sha).
2. **File check** per ticket — ticket names a specific file/component/route → confirm it still exists there. Renamed or removed → `stale`.
3. **Preliminary diagnosis** per surviving ticket — where the symptom likely manifests, a rough guess at subsystem/code-area, and whether it reads as high-impact/visual (UI-visible, blocks a flow) vs. low-impact/cosmetic. This is a hypothesis to drive grouping only — it is NOT the authoritative root-cause call; Step 3 still re-diagnoses on the main model before any fix, per the hack gate.

Return format: one line per ticket — id, verdict (`confirmed-open`/`auto-closed`/`stale`), subsystem guess, impact guess, one-line hypothesis. No commentary.

Apply auto-closed/stale verdicts to `backlog.md` immediately, don't queue them. Surface the prunes to the user before the batch proposal below.

## Step 1 — Group into batches, then confirm with the user

Using the subagent's subsystem/impact tags, group `confirmed-open` tickets into batches:

- **Batches group by shared code-area/subsystem** — closely-related bugs and improvements can mix in the same batch (a batch is a work session, not a commit; each item inside it still gets its own diagnose → fix → commit).
- **Order batches high-impact/visual-first**: anything `#blocking` or user-facing/visual leads, functional-broken next, low-impact/cosmetic last.
- **Within a batch**, sequence items by dependency (fixing A precedes B if A touches B's area).

Surface the proposed batches (labels + member ids + one-line reason each) via `AskUserQuestion` — confirm / reshuffle items between batches / reorder / drop. Do not start fixing until the batch plan is confirmed.

## Step 2 — Write the state file

`.polish-state.json`, project root:

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

Per-item status ladder: `queued → diagnosing → fixing → verifying → shipped-<commit> | blocked | deferred`. Per-batch status ladder: `queued → active → awaiting-dogfood → done`. On any new turn, **read `.polish-state.json` first** and resume at `currentBatch`/`currentIndex` — a batch found `awaiting-dogfood` means stop and re-ask the user before touching the next batch, don't silently continue.

## Step 3 — Per-bug loop (the core, repeat for the item at `currentIndex` within `currentBatch`)

1. **Restate** — id + observable symptom + evidence, one plain-language line.
2. **Diagnose.** Locate where it manifests, trace upstream to where the value/behavior is **FIRST wrong**. Prefer the upstream fix over patching the boundary; flag any fix that infers intent from surface form (regex on formatting, magic-string heuristic, special-case) as a smell. State the root cause in ONE line before writing code.
3. **Hack gate (commit precondition).** Classify: `[root-cause]` proceeds. `[symptom-patch]` or `[heuristic]` — surface the tradeoff (the clean upstream fix vs. why this is a band-aid) and proceed only on explicit user opt-in. Record the choice.
3b. **Felt-impact fork.** If the fix forces a choice the user will *feel*, with no strictly-better option, surface it as a fork (`AskUserQuestion`: options + plain-language tradeoff + recommended default) and apply their pick — don't decide a felt product call for them. A bug with one correct outcome isn't a fork; just fix it. Form: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`.
4. **Implement.** Root-cause reasoning stays on the main model; delegate mechanical edits to Sonnet/Haiku. Touch only this item's files.
5. **Verify.** `/build-review`'s browser engine, three-signal gate, scoped to this bug — its repro scenario plus a blind problem-resolution check — plus typecheck/relevant tests. Not fixed until its specific symptom is gone on screen.
6. **Commit atomically + sync.** One id → one commit; body carries the one-line root cause + classification tag. Update `backlog.md` (`T-N`/`DF-N` status). Named external source → tick its item (fetch first, edit in place, non-destructive). Set `.polish-state.json` to `shipped-<commit>`, advance `currentIndex`.

## Step 3b — Batch pause (after the last item in the current batch)

When `currentIndex` reaches the last item of `currentBatch` (shipped or blocked), stop — don't roll into the next batch's first item in the same turn.

1. Set the batch's status to `awaiting-dogfood` in `.polish-state.json`.
2. Report the batch: ids shipped (with commits), ids blocked (with reasons).
3. Tell the user this batch is ready for their own hands-on dogfood, and ask (`AskUserQuestion`) whether to continue to the next batch, pause here, or re-open something in this batch.
4. Only on "continue" — set the batch to `done`, advance `currentBatch`, reset `currentIndex` to the next batch's first item, and proceed.

## Step 4 — Escalate, don't stall

An item still failing after **3 fix attempts** → `blocked`, one-line reason, leave it in place, **continue the batch**. A blocked item never halts the others.

## Step 5 — Wrap

Past the end of the last batch: report by batch, then by id within it — **shipped** (with commits), **blocked** (with reasons), **deferred/dropped**. Confirm `backlog.md` (and the named external source, if any) are in sync. Delete `.polish-state.json` on a clean drain; leave it if any item is `blocked`, so a follow-up run resumes the unfinished set.

## Integration notes

- **Ids are load-bearing.** State the id every turn (start, fix, resolve). Never renumber `T-n`/`DF-n` or the external source's ids.
- **`backlog.md` is status truth; the external source is intake.** Tick the external source only when an item ships.
- **The anti-hack gate is shared.** Steps 2–3 + the commit tag are the same lens `/build-review`'s structural pass uses: a fix commit states where behavior is FIRST wrong plus `[root-cause]`/`[symptom-patch]`/`[heuristic]`; the latter two need explicit user opt-in, never a silent band-aid.
- **Brain auto-save** fires per commit — a non-obvious root cause found during diagnosis qualifies.
