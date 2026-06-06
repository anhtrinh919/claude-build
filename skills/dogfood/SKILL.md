---
name: dogfood
description: >
  Tests a web app by actually using it via /browse — never by reading code. Two modes:

  Feature check (default after any implementation outside /build): scoped to what was
  just built. Derives test scenarios from context (spec, git diff, or described feature).
  Runs a fix-and-verify loop. Blocks the agent from reporting the feature as done until
  it passes. Auto-triggered whenever implementation finishes outside the /build pipeline —
  use when "done", "just built X", "just fixed X", or implementation clearly just ended.

  Full sweep (explicit request): tests the entire app across all user flows. Finds bugs,
  fixes them in source code, commits each fix atomically, re-verifies. Produces a health
  score and ship-readiness summary. Use when "qa the app", "test everything", "full sweep",
  "find all bugs".

  NOT used inside the /build pipeline — that uses /review. Never report-only.
user-invocable: true
argument-hint: "[feature description, 'full' for a full sweep, or leave blank to auto-detect]"
---

# /dogfood

You just built something. Now use it as a user would — via a real browser, not by reading the code. This skill is the verification gate for all implementation work outside the `/build` pipeline.

**Two modes. Never mix them.**

| Mode | Scope | When |
|------|-------|------|
| Feature check | What just changed | After implementing a feature or fix outside /build |
| Full sweep | Entire app | Explicit request: "test everything", "qa the app" |

Both modes run a fix-and-verify loop. Neither is report-only.

**Hard rule:** Use `/browse` for all browser interactions. Never use `mcp__claude-in-chrome__*` tools.

---

## Step 1 — Determine mode

**Full sweep** if the user said: "full", "sweep", "everything", "qa the whole app", "find all bugs", or similar broad language.

**Feature check** if:
- There's a recent implementation context (spec files, git diff, described feature, recent commit)
- User said "test what I built", "does this work", "dogfood X", or described a specific thing
- Implementation just finished in this session

**If genuinely ambiguous**, ask once:

> "Should I test just what was just implemented, or do a full sweep of the entire app?"

---

## Step 2 — Find the running app

Before any browser interaction:

1. `lsof -i :3000 -i :3001 -i :5173 -i :8080 -i :4000 -i :8000 2>/dev/null | grep LISTEN`
2. If running — use that port.
3. If not — look for a start command in `package.json` (`scripts.dev` or `scripts.start`) or `CLAUDE.md`. Start it in the background; wait up to 15s for a response.
4. If still not found — ask once: "What URL is the app running on?"

---

## Feature check mode

### Determine the original problem AND what was built

In order:
1. Argument passed to `/dogfood` — use it directly. Try to extract both the problem and the solution from it.
2. Spec files — read `specs/*/requirements.md` (most recent dated directory). Extract the **user goal** (what couldn't they do before?) AND the feature summary.
3. Git diff + commit messages — `git diff HEAD~1 --stat` and `git log -1 --format=%B`. Commit messages often state the problem; the diff shows the solution.
4. If the original problem is not clear — ask once: "What problem were you trying to solve? (Not what you built — what was the user struggling with before this?)"

Write **two distinct sentences**:
- **Original problem:** "Before this change, a user [couldn't do X / kept hitting Y / had to work around Z]."
- **What was built:** "We added [the implementation]."

If those two sentences read the same, you're verifying the implementation, not the problem. Stop and re-derive the problem.

### Derive test scenarios — from the problem, not the implementation

Start from the **original problem** sentence. Write: "A user with this problem would naturally try to..."

Derive 2–4 scenarios from what such a user would *attempt* — not from what was coded.

For a bug fix: scenario 1 reproduces the user's situation that caused the bug (the workflow they were doing when they hit it), not just the technical failure mode.
For a new feature: scenario 1 is "user arrives with the problem, tries to solve it naturally" — not "user finds the new feature and uses it correctly."

**Classify each:**
- **Simple**: navigate → verify something is present/absent/correct.
- **Flow**: action → verify intermediate state → action → verify outcome.

For a bug fix: scenario 1 reproduces the original broken behavior and verifies it no longer occurs.
For a new feature: golden path is scenario 1; one edge case (empty state, validation failure, error state) is scenario 2 if applicable.

Document before executing:
```
Scenario 1 (Flow): Click "Add to cart" → cart badge increments → navigate to /cart → item listed with correct price
Scenario 2 (Simple): Navigate to /cart with empty cart → empty state message is shown
```

### Execute via /browse

Apply **Engine 1** from `${CLAUDE_PLUGIN_ROOT}/skills/_shared/browser-review-engine.md`. Pass the scenarios you derived above (verbatim, classified Simple/Flow) and the running app URL. The Sonnet subagent runs one clean pass and reports PASS/FAIL per scenario.

### Naive reviewer pass (mandatory before gate)

Apply **Engine 2** from `${CLAUDE_PLUGIN_ROOT}/skills/_shared/browser-review-engine.md`. Pass:
- The **Original problem** sentence (verbatim)
- The memorable-thing line from the project's `design-brief.md` `## Design intent` section if a design brief exists for the area being tested (look in `specs/*/design-brief.md`); otherwise omit
- The running app URL + login credentials if any

Naive findings merge into your bug/UX issue lists — they are first-class signals, not "nice to have."

### Gate — three signals

Apply **Engine 3** from `${CLAUDE_PLUGIN_ROOT}/skills/_shared/browser-review-engine.md`. The feature is not done until all three signals pass.

Do not output "done", "complete", "feature implemented", or equivalent until the gate clears. This applies even if the user said "finish up" earlier — that instruction predated the gate result.

**Severity-specific surfacing (extends the engine's gate handling):**

- **Signal #2 fails (naive can't accomplish the goal):** treat as a HIGH issue. Apply Engine 4's fix loop. If still failing after the cap, surface: "Naive reviewer still can't solve the original problem after 3 fix attempts. Accept as-is, or keep working?"
- **Signal #3 fails (signals #1 and #2 passed but friction is rough):** surface as a user decision, not an auto-fix: "Functionally complete. Naive reviewer flagged: [list]. Fix now or accept and move on?"

**Avoid double-counting with the fix loop.** The naive reviewer runs *after* scenarios pass — by then per-scenario fixes have addressed functional bugs. Naive findings should be NEW (problem-recognition gaps, friction the scenarios didn't probe). If naive surfaces something the scenario fix loop already addressed, treat it as a regression check, not a new bug.

### Synthesis — Sonnet subagent writes the gate report

Before entering the fix loop or surfacing any decision to the user, **delegate the write-up to a Sonnet subagent**. The main agent has already collected raw findings from Engines 1 and 2 — turning them into a structured, severity-tagged report is execution work, not reasoning work, and it shouldn't burn Opus tokens.

**Spawn pattern.** `Agent` tool with `subagent_type: general-purpose` and **`model: "sonnet"`**.

**Brief template:**

```
You are synthesising browser-test findings into a gate report. No tools — text only.

Original problem: <problem sentence verbatim>
Scenarios run (with PASS/FAIL): <Engine 1 raw output>
Naive reviewer report: <Engine 2 raw output, verbatim>

Produce a report in this exact shape:

# Dogfood — Feature check — [feature one-liner]

**Gate:** PASS / FAIL  (signal #1 / #2 / #3 status)

**Findings (severity-tagged per Engine 3 mapping):**
- [HIGH/MEDIUM/LOW] [one-line description, citing scenario # or naive-reviewer moment]
- ...

**Next action recommended for main agent:**
- If gate PASS: "Clear to report complete."
- If signal #1 fail: "Enter Engine 4 fix loop on failing scenarios."
- If signal #2 fail: "Enter Engine 4 fix loop on naive-reviewer goal failure."
- If signal #3 fail (only): "Surface to user — fix now or accept and move on?"

No prose outside this template. No commentary. No fix suggestions — that's the main agent's job.
```

Main agent reads the returned report and acts on the "Next action recommended" line. The report itself is what the user sees.

### Fix loop

Apply **Engine 4** from `${CLAUDE_PLUGIN_ROOT}/skills/_shared/browser-review-engine.md`. Code edits stay inline on the main agent; re-verifications spawn fresh Sonnet subagents per Engine 1 (scenario fail) or Engine 2 (naive-reviewer fail). 3-attempt cap per failing signal.

**After the fix loop terminates** (gate cleared or cap hit), spawn a final Sonnet subagent with the same brief shape above plus the fix-loop log, to produce the closing report. Main agent uses it for the final user-facing message.

---

## Full sweep mode

### Map the app

Navigate to the app root. Discover all major sections and user flows:
- Check the nav menu, sitemap, or `product.md` for the screen inventory.
- Build a list of flows: authentication, core CRUD operations, key user journeys.

### Test tier

Default: **Quick** (critical and high severity only).

User can specify:
- `standard` — includes medium severity
- `exhaustive` — includes cosmetic issues

### Execute via /browse

Apply **Engine 1** from `${CLAUDE_PLUGIN_ROOT}/skills/_shared/browser-review-engine.md` with one extension for full-sweep mode: instead of a fixed scenario list, pass the full flow list you mapped above plus the chosen tier. Brief the subagent to classify each issue as Critical / High / Medium / Low. Quick tier still reports medium/low for visibility — the main agent decides what to act on per tier.

(Full sweep does NOT run the naive reviewer — there's no single user-goal sentence in app-wide testing. Engines 2 and 3 are feature-check-only.)

### Synthesis — Sonnet subagent writes the pre-fix triage

Before the fix loop runs, **delegate the triage write-up to a Sonnet subagent** — same reasoning as Feature check mode: turning raw Engine 1 findings into a severity-tagged, deduplicated list is execution work.

**Spawn pattern.** `Agent` tool with `subagent_type: general-purpose` and **`model: "sonnet"`**.

**Brief template:**

```
You are triaging full-sweep browser-test findings. No tools — text only.

Flows tested: <list>
Tier: <quick / standard / exhaustive>
Raw findings (per flow, with PASS/FAIL and one-line evidence): <Engine 1 raw output, all flows>

Produce a triage report in this exact shape:

## Triage — pre-fix

**Counts:** N critical, N high, N medium, N low.

**Issues (grouped by severity, deduped where the same root cause hit multiple flows):**
- [CRITICAL] [flow: X] [one-line description]
- [HIGH] [flow: Y] [one-line description]
- ...

**Recommended fix order (main agent):**
[ordered list of issue IDs to attack first — critical → high → tier-included medium/low]

No prose outside this template. No fix code — that's the main agent's job.
```

Main agent reads the triage and runs the fix loop in the recommended order.

### Fix loop

Apply **Engine 4** from `${CLAUDE_PLUGIN_ROOT}/skills/_shared/browser-review-engine.md`, scoped to critical/high issues per the chosen tier. After 3 attempts on any single issue, log it as unresolved and move on (full sweep never blocks).

### Health report — Sonnet subagent writes it

After the fix loop terminates, spawn a final Sonnet subagent to produce the Health report. Same pattern as the triage step (`subagent_type: general-purpose`, `model: "sonnet"`). Pass it: the original triage, the fix-loop log (which issues fixed with commit SHAs, which unresolved), and the optional post-sweep re-verify results.

**Required report shape:**

```markdown
# Dogfood — Full sweep — [date]

**Health:** [before score] → [after score]  (N critical, N high, N medium, N low found)

**Fixed:** [list of issues with commit SHAs]
**Unresolved:** [any issues not fixed, with reason]
**Ship-readiness:** [Ready / Needs attention — one sentence]
```

Main agent forwards the report to the user verbatim.

---

## Ground rules

- `/browse` only — never `mcp__claude-in-chrome__*`.
- **Model split.** Browser passes (Engines 1, 2) and report synthesis (gate report, full-sweep triage, Health report) all run on **Sonnet subagents**. Code fixes (Engine 4) and final user-facing decisions stay on the **main agent**. The only reasoning that should burn main-agent tokens is the actual code-edit work.
- Fix root causes, not symptoms. No surface-patching.
- Commit each fix atomically before re-verifying.
- Feature check is a gate. Do not report completion until it passes or the user explicitly accepts.
- Full sweep produces a health report but does not block.
- Do not expand scope during fixes. If a fix requires touching something outside what was built, surface it instead of doing it silently.
