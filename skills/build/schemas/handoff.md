# handoff.md — Schema

> Agent context — not for human reading.
>
> Project-root file. Written at the end of every phase. Captures **small context a fresh session would miss on init** — things that are not big enough to be a `mission.md` update, not a learning for `WIKI.md`, not a decision for `docs/decisions.md`, and not a tweak for `CLAUDE.md`, but would still slow a fresh session down if it had to re-derive them.

## Disambiguation from `handover.md`

There are two similarly-named files in an SDD project — they are different:

| File | Where | Who writes | Purpose |
|---|---|---|---|
| `specs/YYYY-MM-DD-<feature>/handover.md` | spec dir | `/frontend` Stage 5 | Frontend → backend frame index (which design node maps to which spec screen). Per-phase. |
| `handoff.md` (this schema) | project root | `/spec` Mode 3 at end of each phase | Session → session. Notes the next fresh agent session would otherwise miss. Append-only — one section per phase. |

Same project, different files, different audiences. Don't merge them.

## When to write

`/spec` Mode 3 (between-phase docs update) appends one `## Phase <N> — Handoff Notes` section after Step 5 (architecture update) and before Step 6 (changelog). Exactly one section per completed phase. Append-only — never rewrite or overwrite prior sections.

## What goes in (and what does NOT)

**In:**
- Quirks discovered during this phase that a future session would re-discover painfully (e.g. "the dev server needs `DB_PATH=...` set explicitly when started from a non-default working dir").
- Manual setup steps an operator did mid-phase that aren't in `README.md` setup (e.g. "created a Cloudflare D1 binding via dashboard — not via wrangler").
- Project-state quirks that aren't permanent (e.g. "Phase 2 left `seed.ts` referencing a fixture that gets removed in Phase 3 — don't be surprised by the temporary mismatch").
- Things that almost-but-didn't make it into `tools.md` (a path, an env var) because they were one-off in scope but worth a note.
- Operator-side context: which device the user dogfooded from, what URL pattern worked, anything they verbally said about how they want to use the app going forward.

**Not in:**
- Anything that belongs in `mission.md`, `tech-stack.md`, `roadmap.md`, `WIKI.md`, `docs/architecture.md`, `docs/api.md`, `docs/decisions.md`, `CHANGELOG.md`, `CLAUDE.md`, or `polish.md`. If a note belongs in one of those, put it there — duplicates rot.
- Implementation play-by-play. Git log has that.
- Learnings for `WIKI.md`. `WIKI.md` is "what was learned across phases that future agents need." `handoff.md` is "what would a fresh session today re-derive painfully."
- Anything the user already saw in the phase-complete `/eli` summary unless it has a "but watch out for" twist.

If a section comes out with zero items, write `*(No fresh-session gotchas this phase.)*` rather than skipping the header — the empty section is a positive signal, not a forgotten section.

## Template

```markdown
> Agent context — not for human reading.

# Project handoff notes

Per-phase notes for the next fresh agent session. Append-only — newest section at the bottom. Read this file (last section first) when picking up a project after a break or in a new session.

---

## Phase 1 — Handoff Notes  *(written YYYY-MM-DD by /spec Mode 3)*

### What a fresh session might re-derive painfully
- *(One line each. Two to six items typical.)*

### Operator-side context
- *(How / where / on what device the user dogfooded. Anything they said about how they want to use the app from here.)*

### Things that almost made it into permanent docs
- *(Notes that don't quite belong in mission/tech-stack/wiki/decisions — but are worth a glance for the next session.)*

---

## Phase 2 — Handoff Notes  *(written YYYY-MM-DD by /spec Mode 3)*

[same shape, appended below Phase 1]
```

## Read protocol

`/build`'s project state prime (next-feature path) reads the **last section only** of `handoff.md` — not the whole file. Older phases' handoffs are historical; the current session needs the most recent one.

If the file does not exist yet (Phase 1 not yet completed), skip silently.

## Rules

- Append-only. Never edit or delete a prior phase's section.
- One section per phase, written by `/spec` Mode 3 exactly once.
- Write to the section directly — do not maintain a "drafts" pre-section.
- Read the last section on fresh-session init via `/build`'s project state prime.
- Empty section gets an explicit "no gotchas" line, not silence.
