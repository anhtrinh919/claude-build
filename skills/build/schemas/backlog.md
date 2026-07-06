# Backlog — [Project Name]

> Agent context — not for human reading.

Short-term task lake. This is NOT the roadmap (committed phases), NOT `CLAUDE.md ## Project directives` (durable preferences), and NOT `WIKI.md` (learnings). It is transient: things to do that don't belong in the current phase's spec.

## Numbering & communication — read this first

**Every item has a stable ID, and you ALWAYS tell the user the ID.** This is how you and the user stay grounded — so "is DF-3 fixed?" and "roll T-7 into this phase" mean the same thing to both of you.

- `DF-N` — **Reports** (threaded dogfood bugs/feedback).
- `T-N` — **Tasks** (flat: roll-in / polish / side).
- IDs are monotonic per project and **never reused** — pick the next number above the current max of that prefix. To find it, scan this file.
- **When you file an item, say the ID in chat** ("Filed as DF-3", "Noted as T-7"). **When you resolve or discuss one, refer to it by ID** ("DF-3 fixed — toast added"). Never discuss a backlog item without its ID.

**Status markers:** `open` · `fixed YYYY-MM-DD` · `wontfix` · `rolled-into-Phase-N` · `done YYYY-MM-DD` · `dropped YYYY-MM-DD`

---

## Reports (dogfood — threaded)

Bugs and feedback the user raises while dogfooding a phase. **The user reports in chat; you file, fix, and reply in chat — the user never edits this file.** Each report is one threaded entry: the user's report (R1), your response, then any follow-up rounds (R2, R3…). One terse line per round — this is a ledger, not a transcript.

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

Small changes or additions that could be folded into an upcoming phase without scope-creep. `/build-spec` (phase mode) reads this at phase start and surfaces any items that fit the incoming scope.

Format: `- [T-N] [roll-in] YYYY-MM-DD [description] — [status]`

- *(none yet)*

---

## Dogfood polish

UX nits, visual rough edges, or small copy fixes surfaced by dogfood passes (blind or guided) but deferred because they are LOW severity and non-blocking. `/build-review` (pipeline-review mode) appends here instead of triggering the auto-fix loop for these.

Format: `- [T-N] [polish] YYYY-MM-DD [Phase N] [description] — [status]`

- *(none yet)*

---

## Side tasks

One-off user requests, deferred asks ("later, can you…"), or out-of-band tasks the user raised mid-build that don't belong in the current phase's spec. Append the moment the user defers something, then confirm in one line **with the ID** ("Noted as T-7"). Read at the start of any dogfood-polish or side-task session to stay oriented.

Format: `- [T-N] [side] YYYY-MM-DD [description] — [status]`

- *(none yet)*
