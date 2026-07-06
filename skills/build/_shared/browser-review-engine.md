# Browser Review Engine — the single browser-review engine, used by /build-review (both modes)

The browser-driven review pattern used by `/build-review` in **both** modes — `pipeline-review` (inside `/build`) and `standalone-dogfood` (post-implementation gate outside `/build`). It runs scenarios via a Sonnet subagent, runs a blind naive-reviewer subagent, and gates completion on three signals. There is exactly one such engine (this file); both modes call it and differ only in scope and terminal write — `pipeline-review` layers shell-regression + outcome-card grading on top. (Historically pipeline review ran its own duplicate dogfood session; that duplication is gone.)

The caller supplies the **scope** (which scenarios, which user goal); this reference owns the **engine** (how subagents are briefed, what the gate checks, how the fix loop iterates).

**Callers must run in the main session.** Every engine spawns subagents, and subagents cannot spawn subagents — a caller running as a subagent silently breaks every engine (one-level nesting rule, `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md`).

## Why Sonnet for the subagents

Browser-driven scenario execution and naive first-impression review are both *execution* work, not architectural reasoning. Pinning Sonnet keeps cost predictable when the caller (`/build-review` standalone-dogfood mode) is invoked from Opus. Fix work — code edits, root-cause analysis — stays inline on the main agent (whatever model the caller runs on).

## Why fresh subagents on every iteration

Once a subagent has seen a discrepancy, it's no longer a clean read. Both the scenario subagent (after a fix) and the naive reviewer (after a fix) must be re-spawned fresh — never reused. Each iteration is a new instance with the same brief shape but the updated app state.

---

## Engine 1 — Scenario subagent

Browser execution of pre-defined scenarios. The caller supplies the scenario list; the engine runs them one clean pass each via `/browse`.

**Spawn pattern.** `Agent` tool with `subagent_type: general-purpose` and **`model: "sonnet"`**.

**Brief template — caller fills the slots:**

```
You are running a browser-driven scenario pass. Tools: /browse only. No source access, no file edits, no fixing.

Running app URL: <URL>

Scenarios (run each one clean pass — no retries, failure is a signal):
<list of 2–N scenarios, classified Simple or Flow>

Per-scenario rules:
- Simple: navigate → verify → screenshot.
- Flow: each action is a discrete step. Verify the expected intermediate state after each action before proceeding. If an intermediate state is wrong, stop that scenario immediately — record the failure, move to the next. Do not improvise a workaround.

Output per scenario — one block per scenario, no prose outside these blocks:

Status: PASS | FAIL | UNEXPECTED | MISSING
- PASS: outcome matches the story exactly.
- FAIL: expected: <one line> / seen: <one line>
- UNEXPECTED: outcome is technically correct but the app did something the story does not describe (extra UI, unsolicited state changes, side effects the user didn't trigger) — describe: <one line>
- MISSING: a step the story requires is simply absent from the UI — nothing to click, no field, no screen — describe: <one line>
Screenshot: <file path saved to /tmp/ — never embed base64>
```

The subagent reports back; the main agent owns the fix loop (see Engine 3).

---

## Engine 2 — Naive reviewer subagent (blind first-impression)

A subagent that *literally cannot* see the implementation tries to accomplish the user's goal using only what's on screen. Catches what persona-roleplay alone misses — you built this and can't unsee it.

**Spawn pattern.** `Agent` tool with `subagent_type: general-purpose` and **`model: "sonnet"`**.

**Brief template — caller fills the slots:**

```
You are a user with this problem. You don't know what was just built or how. Open the app and try to accomplish your goal using only what you see on screen.

Persona (if provided — adopt it fully, including viewport): <persona line from the caller, or omit — default is a first-time user who has never seen this app>
Your goal: <user-goal sentence — verbatim, nothing else from the spec>
Memorable thing the experience should leave you with (if provided): <memorable-thing line from design-brief.md ## Design intent, or omit if none>
In scope this session (if provided): <scope-fence line — the capability that is live now, and which surfaces are intentional not-yet-wired placeholders to note but never judge as broken; omit if the caller passes none>

Running app URL: <URL>
Login (if needed): <credentials from .env / CLAUDE.md, or omit>

Tools: /browse only. No file reads, no git, no spec access, no handover.md.

First-impression report — answer all of the following:

1. Did you accomplish your goal? Yes / Partial / No.
2. If a memorable-thing line was provided: did the experience leave you feeling that? Cite specific moments.
3. What told you it worked (or didn't)? If you can't point to specific on-screen feedback, the answer is "I'm not sure."
4. Where did you hesitate, get confused, or guess? Cite specific moments — "I didn't know whether to click X or Y", "I expected feedback after Z but got nothing", "the label said A but I thought it would do B".
5. Anti-pattern scan — explicitly check each:
   - Did you re-read any text twice to understand it?
   - Was there any moment you didn't know what to click next?
   - Did anything feel "broken" even if it technically worked? (flicker, weird timing, dead space, ghost element)
   - Were 2+ similar-looking elements competing for attention?
   - Was success communicated unambiguously, or did you have to infer it?
   - Was failure communicated with a clear recovery path, or just an error?
   - Did anything use developer/implementation language? ("submitted", "request received", "200 OK", raw IDs, status codes, jargon)
   - Was the visual hierarchy obvious without thinking?
   - On mobile-sized viewport (390×844): did anything overflow, get squished, or hide content?
```

**Hard rule — what the caller MUST NOT brief:** `requirements.md` (full), `plan.md`, `handover.md`, the diff, what was built, the implementation conversation, or any hint about which feature is being tested. The reviewer must be blind to the solution to give a real first-impression read. The optional in-scope line is the *only* boundary the caller may pass — it names which surfaces are live vs. deliberate placeholder so the reviewer doesn't score unbuilt scaffolding as broken; it may reveal *whether a surface is on the exam*, never *how* a feature works.

Pass ONLY: the optional persona line, the user-goal sentence, the optional memorable-thing line, the optional in-scope/scope-fence line, the URL, login credentials. Nothing else.

**Persona variants (optional).** A caller may run Engine 2 several times with different persona lines (e.g. first-timer desktop, impatient mobile, returning user). Each variant is a separate fresh subagent with the identical blindness rules; the persona line changes the lens, never the information available. Callers that pass no persona get the default first-time reviewer.

---

## Engine 3 — The three-signal gate

**Engine 1 status routing:** `FAIL` triggers the fix loop. `UNEXPECTED` and `MISSING` are surfaced in the consolidated report as informational findings (severity `LOW` unless the caller escalates them) — they do NOT trigger a fix-loop iteration. They are signals for the outcome-grading pass and the naive reviewer, not blockers.

The work is not done until **all three** are true:

1. **Functional** — every scenario from Engine 1 returns PASS, UNEXPECTED, or MISSING (no FAILs).
2. **Problem-resolved** — the naive reviewer (Engine 2) reports "Yes" to the goal AND can point to specific on-screen feedback that told them so. If a memorable-thing line was passed, the reviewer's answer to question 2 is at least "yes, the experience felt that way" or equivalent.
3. **No severe friction** — the naive reviewer reports no moments of "I didn't know what to do" or "I had to infer success" on the primary path.

**Severity tagging from naive findings (caller uses these to merge with its own issue lists):**

- "I couldn't accomplish the goal" → **HIGH**
- "I'm not sure if it worked" on the primary goal → **MEDIUM** minimum
- "I had to guess what to click" / "I re-read text twice" / "no clear feedback after action" → **MEDIUM** UX issue
- Memorable-thing failure ("the experience didn't feel that way") → **MEDIUM** UX issue
- Anti-pattern hits (developer language, weak hierarchy, etc.) → **LOW** unless they materially blocked the goal

---

## Engine 4 — Fix loop (caller owns, engine prescribes shape)

When the gate fails, the main agent (NOT a subagent) owns the fix loop. Code edits stay inline on whatever model the caller runs — no fix work is delegated to Sonnet.

**Per failure (whether scenario fail or naive-reviewer fail):**

1. Identify the root cause from the browser evidence (what was visible vs. what was expected).
2. Fix in source — root cause only, inline (main agent). Do not refactor, do not expand scope.
3. Commit atomically: `git commit -m "fix: [one-line description]"`.
4. Re-verify by spawning a fresh subagent of the matching engine — Engine 1 if a scenario failed, Engine 2 if the naive-reviewer signal failed. Same brief shape; fresh instance per re-verify.
5. Pass → move to next failure. Fail → counts as iteration 2.

**Re-verification scope rule.** On re-verify after a fix, scope the subagent to the failing scenario or the failing naive-reviewer goal — not the full pass. The main agent decides when to re-run the full set (typically only after the last fix lands or when a fix could plausibly have regressed elsewhere).

**Skip the naive reviewer on iterations 2+ unless the naive-reviewer signal itself failed.** If only Engine 1 (scenarios) is failing, re-verify only those scenarios. Spawning a fresh naive reviewer on every fix iteration is wasted cost — it didn't fail.

**Cap: 3 iterations total per failing signal.** After 3, surface to the user as a binary:

```
[Caller name]: [signal name] still failing after 3 fix attempts.

Expected: [one line]
Seen: [one line]

Accept as-is, or keep working?
```

Wait for the user. No further attempts unless they explicitly ask.

---

## Caller responsibilities (what the engine does NOT cover)

`/build-review` is the caller in **both** modes. **standalone-dogfood** runs outside `/build`, has no `.build-state.json`, and derives scope from "what just changed" (the bullets below). **pipeline-review** runs inside `/build` and derives scope from the phase spec (`validation.md` stories + `outcome-card.md`) instead, but drives this same engine and adds shell-regression + card grading on top.

- **Scope derivation.** In standalone-dogfood mode, `/build-review` derives 2–4 scenarios from "what just changed", then calls Engine 1 with that scenario list.
- **The user-goal sentence.** `/build-review` (standalone-dogfood mode) derives it from the original problem and passes it verbatim to Engine 2.
- **Issue aggregation and reporting.** The caller owns its report shape (severity tables, health score, etc.). The engine produces raw findings; the caller merges them into its report.
- **Iteration counter persistence.** `/build-review` (standalone-dogfood mode) tracks fix-loop iteration in-conversation only — no state file.
