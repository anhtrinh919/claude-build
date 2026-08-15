---
name: backend
description: Backend implementation for /build. Reads requirements.md, plan.md, and the phase design source, then implements plan.md groups wave-by-wave — topologically sorted, non-overlapping file sets, Opus for architecture and Sonnet for leaf work — then integration-tests every API contract.
---

# /build:backend — backend implementation

Turn the approved phase spec into a working, integration-tested backend — every API contract in `requirements.md`, nothing beyond it. Done when every group's verify script is green and every contract is exercised with real requests.

Enter through `/build` (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`). Plain language to the user (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Mode | Model | Mechanism | Inputs | Outputs | Terminal step |
|---|---|---|---|---|---|
| single | Opus + wave-dispatched Opus/Sonnet agents | inline, in the `/build` session | `requirements.md` · `plan.md` · `design-tokens.css` · the design source | implemented backend, integration-tested against every API contract | **none** — the orchestrator runs backend-compliance and writes `backend-complete`; on fail it rolls back to `design-complete` and re-runs this skill |

This session owns wave dispatch, interface cross-checks, commits, and integration testing (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md`, Rule 7 governs dispatch).

Brain: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md`, `$AGENT=backend`, `$TAGS` from `tech-stack.md`. **Friction trigger:** a group needed a meaningfully different second approach — one entry per group. **Phase-wrap trigger:** once, after Step 3.

## Inputs — find and read first

From the most recent `specs/YYYY-MM-DD-[feature]/`: `requirements.md` (the contract), `plan.md` (build order), `design-tokens.css` if `ui: true`, and the **design source** named by `mission.md ## Design Tool` — on `claude-code` the mockups in `specs/<phase>/mockups/` (they ARE the design) plus `specs/<phase>/design-notes.md` if design wrote one; on `external` the `handover.md` screen→image index plus the images it points to.

Missing `requirements.md`, `plan.md`, or (on `ui: true`) the design source → stop: "No phase spec or design found — run `/build:spec` and `/build:design` first." **`phaseCeremony: "narrow"` exempts the design half** — design never ran, so the source is prior phases' mockups plus the existing `design-tokens.css`. Never invoke `/build:design` on a narrow phase.

**Read `outcome-card.md` as a cross-check.** `requirements.md` wins on conflict, but card→requirements is where an approved outcome silently gets lost, and this is the last builder before review. A group that would visibly diverge from a card outcome → stop it, return `status: needs-decision`.

## Design authority (`ui: true`)

The design wins on visuals — color, spacing, typography, structure. `requirements.md` wins on behavior.

Before any UI group, mirror the **design source**, not the existing codebase: on `claude-code` the mockup is already real code — wire it, do not re-derive it; on `external` open each screen's image via the index. A reusable element is one component with state variants, never duplicated inline. Import `design-tokens.css` globally; tokens are the sole source for color, font, and spacing — never re-extract or approximate.

A screen or state `requirements.md` lists but the design source omits → stop: "Design missing [state] — return to `/build:design`." On a narrow phase that return does not exist, so this is a scoping error — surface it, never guess the screen.

Building from the design source **is** the conformance mechanism; there is no separate post-build visual gate.

## Step 1 — Test plan

Before any implementation code, map every function and path this phase adds or changes — happy path, edge cases, error paths — tagging each `[TESTED]` or `[GAP]`. Any previously-tested path this phase touches needs a regression test that fails without the fix and passes with it.

**Before writing any test, read `${CLAUDE_PLUGIN_ROOT}/skills/backend/references/test-standard.md`.** Non-negotiable: mock only external dependencies, never the mechanism under test — `jwt.verify` and `bcrypt.compare` run for real against a test secret. Every assertion names a specific value. "Tests pass" is not "the feature works".

## Step 2 — Implementation, group by group

Every wave below, and every step after this one, is an internal boundary — cross each one silently, no summary before continuing (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/auto-continue.md`).

Groups build in dependency order. **Resume-idempotency:** before dispatch, run every remaining group's `verify-group-N.sh`. Any that already passes is built and committed — skip it. Log "[N] groups already built, [N] remaining."

### Wave dispatch

Read each group's `Depends on:` field and **topologically sort into waves** — wave 1 has no unmet dependency, wave N+1's dependencies all sit in waves ≤N. Within a wave assign **non-overlapping file sets**; groups sharing a file go to the same agent or a later wave.

| Tier | Criteria | Strategy |
|---|---|---|
| Simple | ≤3 remaining groups, or every wave has 1 agent | Sequential inline — this session implements each group itself, no dispatch |
| Standard / Complex | 4+ remaining groups, ≥1 multi-agent wave | Wave dispatch — parallel agents within a wave, hard barrier between waves |

**Model split.** Opus: data model, migrations, API routes, auth and session logic, anything needing an architectural call. Sonnet: UI components, static config, CRUD wiring to an existing contract, utilities, test helpers. A simple group stays Opus if it shares files with an Opus group in the same wave.

**Every brief carries:** full `requirements.md`, `plan.md`, the design source, `tech-stack.md`; its group numbers only; prior waves' API surface and changed-file list, pasted not referenced; the per-group procedure; the containment string from `subagent-policy.md`; and the stop conditions — 3 failed hypotheses, thrashing, or 2× estimate → return immediately, never surface to the user itself.

**A felt-impact fork returns `status: needs-decision`, never a silent pick** (Rule 7). Invisible plumbing goes to the doc that owns it — `tech-stack.md ## Key Technical Decisions` or `docs/architecture.md`. Implementation notes go to `CHANGELOG.md` at replan.

**Return format, per group, no commentary:**
```
Group N — status: done | blocked | needs-decision
Changed files: [...]
Verify: [verify-group-N.sh result]
Diagnosis: [...]                                    (blocked only)
Fork: [options + tradeoffs + recommended default]   (needs-decision only)
```

**Wave barrier — this session:**
1. `blocked` → retry with a fresh agent carrying the diagnosis (Rule 8), reassign inline, or surface as a last resort. `needs-decision` → surface the fork via `AskUserQuestion`, then re-dispatch a fresh agent with the answer. A felt decision overrides auto-continue.
2. Cross-check interfaces between agents — signatures, API shapes, shared types. Fix mismatches inline.
3. **Commit per group**, this session only, only that group's listed files.
4. Re-run every verify script from scratch. Never trust an agent's claim (Rule 5). Green → next wave.

### Per-group procedure

1. Spec-Light header: `TASK: [group] / ESTIMATE: [time] / VERIFY: [command]`.
2. Write `verify-group-N.sh` before touching implementation.
3. Logic groups: tests before code, per `references/test-standard.md`, confirmed failing first — a test-file comment is not sufficient. Pure-UI and config groups skip; the regression rule still applies to any touched, previously-tested path.
4. Implement the minimum code that passes the tests.
5. Self-verify with `verify-group-N.sh`.
6. **Iron law: no fix without root cause.** Trace the failing path, check what recently changed, form one hypothesis, test it before writing fix code. 3 failed hypotheses, a fix spreading outside the group's scope, 2–3 reverts, or 2× estimate → stop and surface what was tried, what is suspected, the decision needed.
7. Commit with a plain-English summary.

## Step 3 — Integration testing

Test the **full API surface** against every contract in `requirements.md`, all endpoints together. Integration means a real HTTP request → handler → DB → response against the running server. Not browser end-to-end testing.

**Never install or scaffold Playwright, Cypress, Puppeteer, Selenium, or WebdriverIO.** Browser verification is `/build:review`'s dogfood session; a phase genuinely needing a persisted browser suite stops and asks first.

1. Start the dev server; poll until ready, max 30s.
2. For **every** contract: send the exact request and verify the response shape — field names, types, status codes. Trigger every error condition and verify status and message. **Protected endpoints need both 401 cases — missing Authorization header, and a malformed JWT (`Authorization: Bearer invalid.jwt.here`)** — different branches, both required. Verify empty and null input, and concurrency where relevant.
3. Cross-check against the design source — does the implementation match the shape the frontend expects?
4. Failure attempt 1 → diagnose and fix. Attempt 2 → a genuinely different diagnosis. Still failing → surface which endpoint fails, what was tried, what is suspected, the decision needed.

## Completion

Write the phase-wrap brain entry once Step 3 passes. Report: what the backend provides in one sentence, the endpoint count, status (`ready` / `has known issues` / `blocked`), one line on anything fragile.

**Do not** write `step`, and **do not** invoke `/build:review`. Hand straight back.

**Write `currentSubStep: "backend.done"` on clean return — never null.** `step` still reads `spec-complete` or `design-complete` at this moment, because the orchestrator writes `backend-complete` only after its own gate. So `backend.done` is the sole durable record that the groups are built and only the gate is outstanding. Nulling it here makes a finished backend identical on disk to one that never started, and the phase then sits at `spec-complete` looking untouched — with every group committed. The orchestrator clears it when it writes `backend-complete`.

## Ground rules

1. **`requirements.md` is the contract** — nothing more. `plan.md` groups are the build sequence; never reorder or skip them.
2. `tech-stack.md` non-negotiables apply from the first line of code.
3. Architectural review runs in `/build:review`, not here. This skill ends at a working, integration-tested API.
4. **No browser or E2E frameworks, ever, unprompted.**
5. **Worktree isolation is the last resort**, only when non-overlapping file sets are impossible for a wave.
6. This session owns servers, processes, and test runners. Never ask the user to run one.
