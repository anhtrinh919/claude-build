---
name: build-backend
description: >
  Backend implementation skill for the /build SDD stack. Reads requirements.md (API contracts, data model), plan.md (dependency-ordered task groups), design-tokens.css, and the design source — track-dependent: on the claude-code track the mockups in specs/<phase>/mockups/ (they ARE the design, no handover doc) plus docs/decisions.md for non-visual choices; on the external track a bare screen→image index in handover.md pointing at the exported images. Implements plan.md groups wave-by-wave — topologically sorted by Depends-on, non-overlapping file sets per agent, Opus for architectural/data/API groups, Sonnet for leaf/UI/config — then integration-tests every API contract in requirements.md (every endpoint, every error path, both 401 cases). Runs inline in the /build session; wave agents never commit, never start servers, never address the user. Enforces the shared test standard (references/test-standard.md) on every test written. Returns without a terminal state — the /build orchestrator runs backend-compliance and writes backend-complete. Trigger on /build-backend, or when /build reaches the backend step.
---

> **Part of `/build`.** On a resume with an active `.build-state.json`, enter through the `/build` orchestrator — it routes here; don't drive this skill off the state file directly (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`).

# /build-backend — Backend Implementation

Find the most recent `specs/YYYY-MM-DD-[feature]/` directory and read: `requirements.md` (contract), `plan.md` (task groups in build order), `design-tokens.css` if `ui: true`, and the **design source** (track-dependent, `mission.md ## Design Tool`): **`claude-code`** → the mockups in `specs/<phase>/mockups/` (they ARE the design) + `docs/decisions.md` (non-visual design decisions) — there is no handover doc on this track; **`external`** → `handover.md` (the screen→image index) + the exported images it points to. `requirements.md`/`plan.md` missing, or (for `ui: true`) the design source missing → stop: "No phase spec/design found — run `/build-spec` and `/build-design` for this phase first."

## What this skill is

- **Goal.** Turn the approved phase spec into a working, integration-tested backend — every API contract in `requirements.md`, built group by group — nothing beyond it.
- **Read-only cross-check.** Also read `outcome-card.md` — the user's approved outcome, plain language. `requirements.md` is the build contract and wins on conflict, but the card→requirements translation is the one place an approved outcome can silently get lost, and this skill is the last builder before review. A group that would visibly diverge from a card outcome (nothing in requirements.md serves it, or contradicts it) → stop that group, return `status: needs-decision` with the gap. Don't build faithfully-wrong off a mistranslation and leave it for review to catch.
- **How.** Wave-dispatched implementation agents per `plan.md` group; Stage 3 integration-tests the full API surface.
- **Done when.** Every group's verify script is green, every API contract is exercised with real requests (shape, status, every error path), `handover.md`'s expectations are met.

## Invocation contract

| Mode | Model | Mechanism | Inputs | Outputs | Terminal state |
|---|---|---|---|---|---|
| single | Opus (session default) + wave-dispatched Opus/Sonnet agents | inline, in the `/build` session | requirements.md + plan.md + design-tokens.css + design source (`claude-code`: mockups + `docs/decisions.md`; `external`: `handover.md` index + exported images) | implemented backend, integration-tested against every requirements.md API contract | none — this skill returns; the orchestrator runs backend-compliance and writes `backend-complete` (fail → rollback `step` to `design-complete`, re-run this skill) |

This session owns wave dispatch, interface cross-checks, commits, and integration testing. Wave agents never commit, never start servers, never address the user (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md` — Rule 6 governs the dispatch below).

Voice: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`. Brain: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md`, `$AGENT=backend`, `$TAGS` from `tech-stack.md`. **Friction trigger:** a group needed a meaningfully different second approach after code-harness rejected the first — one entry per group, title `Phase <N> backend friction: <issue>`. **Phase-wrap trigger:** once, after Stage 3 passes — title `Phase <N> backend: <summary>`.

---

## Design authority (`ui: true` phases)

The design wins on visuals (color, spacing, typography, structure); `requirements.md` wins on behavior. Before any UI group, read the **design source** (per track, above) and mirror it, not the existing codebase: **`claude-code`** → build each screen from its mockup in `specs/<phase>/mockups/` (real code already — wire it, don't re-derive it); **`external`** → open each screen's exported image via the `handover.md` index. A reusable design element ("Phase Card / Running") is one component with state variants, never duplicated inline per screen. Import `design-tokens.css` into the global stylesheet; tokens are the sole source for color/font/spacing values — never re-extract or approximate. A screen/state `requirements.md` UI Requirements lists but the design source doesn't cover → stop: "Design missing [state] — return to `/build-design`." Building each screen *from* the design source (above) is the conformance mechanism — there is no separate post-build visual gate; design-quality defects are caught upstream in `/build-design`'s gates.

---

## Stage 1 — Test plan

Before writing implementation code: map every function/path this phase adds or changes — happy path, edge cases, error paths — tag each `[TESTED]`/`[GAP]`. Any previously-tested path this phase touches needs a regression test that fails without the fix and passes with it. Pure-UI/pure-refactor phases skip the diagram; the regression rule and the test standard still apply to any touched, previously-tested path.

**Before writing any test**, read `${CLAUDE_PLUGIN_ROOT}/skills/build-backend/references/test-standard.md`. Non-negotiable: mock only external dependencies, never the mechanism under test (`jwt.verify`/`bcrypt.compare` run for real, against a test secret); every assertion names a specific value; edge cases covered; regression tests fail before they pass. "Tests pass" ≠ working feature — run the real feature against real inputs before calling a group done.

---

## Stage 2 — Implementation, group by group

Groups build in dependency order. **Resume-idempotency (not foundation reconciliation — v2 has no background foundation build):** before dispatch, run every remaining group's `verify-group-N.sh`. Any that already passes is already built and committed — skip it. Build the rest normally. Log: "[N] groups already built, [N] remaining."

### Wave dispatch

Map the dependency graph from the remaining groups: read each group's `Depends on:` field, **topologically sort into waves** — wave 1 = groups with no unmet dependency, wave N+1 = groups whose dependencies are all in waves ≤N. Within a wave, assign **non-overlapping file sets** — groups sharing a file go to the same agent or a later wave.

| Tier | Criteria | Strategy |
|---|---|---|
| Simple | ≤3 remaining groups, or every wave has 1 agent | Sequential inline — this session implements every group itself (per-group procedure below), no dispatch |
| Standard / Complex | 4+ remaining groups, ≥1 multi-agent wave | Wave dispatch — parallel agents within a wave, hard barrier between waves |

**Model split:** Opus — data model, schema migrations, API routes, auth/session logic, any group needing an architectural call. Sonnet — UI components, static config, CRUD wiring to an existing contract, utilities, test helpers. A simple group stays with Opus if it shares files with an Opus group in the same wave.

**Per-wave brief, every agent gets:** full `requirements.md` + `plan.md` + `handover.md` + `tech-stack.md`; its assigned group numbers only; prior waves' implemented API surface + changed-file list (pasted, not referenced); the per-group procedure below; **containment — no spawning agents, no `/build*`, no `claude -p`, no commit, no server, don't address the user — return content/paths/status only**; any stop condition (3 failed hypotheses, thrashing, 2× estimate) → return immediately, do not surface to the user itself.

**Felt-impact fork → `status: needs-decision`, never a silent pick.** A UX or performance choice with no strictly-better option, not already settled in `requirements.md`/`plan.md` → stop that group, return the fork: genuine options, each option's one-line plain-language tradeoff, a recommended default. Invisible plumbing the agent decides itself and records in `docs/decisions.md`.

**Return format, per group, no commentary:**
```
Group N — status: done | blocked | needs-decision
Changed files: [...]
Verify: [verify-group-N.sh result]
Diagnosis: [...]                                    (blocked only)
Fork: [options + tradeoffs + recommended default]   (needs-decision only)
```

**Wave barrier (this session):**
1. Collect returns. `blocked` → retry with a fresh agent carrying the diagnosis (policy Rule 7), reassign inline, or surface as last resort. `needs-decision` → surface the fork now via `AskUserQuestion`, then re-dispatch a fresh agent with the answer — a felt decision overrides auto-continue, even mid-phase.
2. Cross-check interfaces between agents (function signatures, API shapes, shared types); fix mismatches inline.
3. **Commit per group** — this session only, plain-English summary, only that group's listed files.
4. Re-run every verify script from scratch — never trust an agent's claim (policy Rule 5). Green → next wave.

Simple tier runs the same per-group procedure sequentially; commit happens inline, no dispatch.

### Per-group procedure

1. Spec-Light header: `TASK: [group] / ESTIMATE: [time] / VERIFY: [command]`.
2. Write `verify-group-N.sh` before touching implementation.
3. Logic groups: tests before code, per `references/test-standard.md`; run and confirm they fail first (implementation absent) — a test-file comment is not sufficient. Pure-UI/config groups skip. Regression tests for any touched, previously-tested path are mandatory regardless.
4. Implement — minimum code to pass tests / deliver the slice.
5. Self-verify: run `verify-group-N.sh`.
6. Red → root cause before any fix. **Iron law: no fix without root cause** — trace the failing path, check what recently changed, form one hypothesis, test it with a targeted check before writing fix code. 3 failed hypotheses, a fix that spreads outside the group's scope, 2–3 reverts, or 2× estimate with no root cause → stop, surface (approaches tried, what's suspected, the decision needed).
7. Commit — plain-English summary, readable in 2 seconds.

---

## Stage 3 — Integration testing

Test the **full API surface** against every contract in `requirements.md` — all endpoints together, not per-task. Integration = a real HTTP request → handler → DB → response, exercised against the running server. NOT browser end-to-end testing.

**Never install or scaffold Playwright/Cypress/Puppeteer/Selenium/WebdriverIO.** Browser verification is `/build-review`'s dogfood session; if a phase genuinely needs a persisted browser-test suite, stop and ask first.

1. Start the dev server; poll until ready (max 30s).
2. For **every** API contract in `requirements.md`: send the exact request specified, verify the response shape (field names, types, status codes); trigger every error condition and verify status + message; **protected endpoints need both 401 cases — missing Authorization header, and a malformed JWT (`Authorization: Bearer invalid.jwt.here`)** — these hit different branches, both are required; verify edge cases (empty/null input, concurrent requests where relevant).
3. Cross-check against `handover.md` — does the implementation match the shape frontend expects? Flag mismatches.
4. Failure attempt 1 → diagnose and fix. Attempt 2 → a genuinely different diagnosis. Still failing → surface: which endpoint fails, what was tried, what's suspected, the decision needed.

---

## Completion

Write the phase-wrap brain entry (once, after Stage 3 passes). Report to the orchestrator: what the backend provides (one sentence), endpoint count (all integration-tested), status (`ready` / `has known issues` / `blocked`), one line on anything fragile.

**Do not** write `.build-state.json`'s `step` — terminal-state ownership belongs to the orchestrator. Null `currentSubStep` on return (it should have carried a `"backend.group-N"` breadcrumb while a group was in flight, for crash-resume). **Do not** invoke `/build-review` — the orchestrator runs the `backend-compliance` gate first and decides: pass writes `backend-complete`; fail rolls back to `design-complete` and re-runs this skill. On return do NOT yield or summarize-and-stop — hand straight back so the orchestrator auto-continues in the same turn (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/auto-continue.md`).

---

## Ground rules

- `requirements.md` is the contract — build what it specifies, nothing more. `plan.md` groups are the build sequence — do not reorder or skip.
- Code-harness gates every group; `tech-stack.md` non-negotiables apply from the first line of code.
- Architectural/structural review is not a backend stage — it runs in `/build-review`. This skill's job ends at a working, integration-tested API.
- No browser/E2E frameworks, ever, unprompted. Backend testing = unit/logic (Stage 1–2) + API-contract integration (Stage 3).
- **Worktree isolation (`isolation: "worktree"`) is the last resort** when non-overlapping file sets are genuinely impossible for a wave — expensive, rare, not a default.
- This session owns servers, processes, and test runners — never ask the user to. The user is the last resort, not the first.
