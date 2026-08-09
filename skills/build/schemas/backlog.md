# Backlog — [Project Name]

> Agent context — not for human reading.

The transient task lake: `DF-N` dogfood threads and `T-N` tasks. Not the roadmap (committed phases), not `CLAUDE.md ## Project directives` (durable preferences). Things to do that don't belong in the current phase's spec.

## Numbering & communication — read this first

Every item has a stable ID. Always tell the user the ID.

- `DF-N` — **Reports** (threaded dogfood bugs/feedback).
- `T-N` — **Tasks** (flat: roll-in / polish / side).
- IDs are monotonic per project and never reused. Scan this file for the current max of that prefix, then pick the next number.
- Say the ID when you file, resolve, or discuss an item ("Filed as DF-3", "DF-3 fixed — toast added"). Never discuss an item without its ID.

**Status markers:** `open` · `fixed YYYY-MM-DD` · `wontfix` · `rolled-into-Phase-N` · `done YYYY-MM-DD` · `dropped YYYY-MM-DD`

---

## Reports (dogfood — threaded)

The user reports in chat; you file, fix, and reply — the user never edits this file.

Format:
```
### DF-N  [short title] — [status]
- R1 (YYYY-MM-DD, user): [what the user reported]
- (YYYY-MM-DD, agent): [what you did / your answer]
- R2 (YYYY-MM-DD, user): [follow-up]
- (YYYY-MM-DD, agent): [...]
```

Resolution routing: fixed → close it (`fixed` status). Deferred (real but not now) → drop a flat **Task** (`T-N [side]`) and note the DF-N it came from. Needs genuinely new work → a `T-N [roll-in]` candidate.

- *(none yet)*

---

## Roll-in candidates

Changes small enough to fold into an upcoming phase without scope-creep. `/build:spec` (phase mode) reads this at phase start.

Format: `- [T-N] [roll-in] YYYY-MM-DD [description] — [status]`

- *(none yet)*

---

## Dogfood polish

UX nits, visual rough edges, or small copy fixes from dogfood passes: non-blocking notes, deferred. The dogfood scout emits no severities — `/build:review` (pipeline-review mode) appends a note here instead of the auto-fix loop, which takes broken items only.

Format: `- [T-N] [polish] YYYY-MM-DD [Phase N] [description] — [status]`

- *(none yet)*

---

## Side tasks

One-off user requests, deferred asks ("later, can you…"), or out-of-band tasks that don't belong in the current phase's spec.

Format: `- [T-N] [side] YYYY-MM-DD [description] — [status]`

- *(none yet)*
