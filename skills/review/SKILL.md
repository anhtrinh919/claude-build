---
name: review
description: 'Code review and functional dogfood. **pipeline-review** is the /build phase gate: automated checks, the `code-reviewer` agent, then a `dogfood` browser walk, with every HIGH and MEDIUM auto-fixed silently. **standalone-dogfood** fires when implementation ends outside /build.'
user-invocable: true
argument-hint: "pipeline-review | standalone-dogfood | [feature description to dogfood]"
---

# /build:review — code review and functional dogfood

Two modes over two shared agents. All review *judgment* lives in the agents: `code-reviewer` (Opus) owns the lens, Fowler, ponytail, and correctness catalog; `dogfood` (Sonnet) owns the blind first-impression and the guided walk. This skill owns orchestration only — what to pass them, how to gate on what they return, the fix loop, the terminal writes.

Enter through `/build` for `pipeline-review` (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`). A quoted string is intent to convey in plain language (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`), never a script, unless marked **exact**.

## Invocation contract

| Mode | Model | Mechanism | Inputs | Outputs | Terminal step |
|---|---|---|---|---|---|
| **pipeline-review** | Opus main; `code-reviewer` + `dogfood` agents; Opus fix leaves | inline | spec dir, `validation.md`, `outcome-card.md`, `requirements.md`, `docs/rejected.md`, running app | review report + silent fixes + dogfood handoff | **`phase-complete`** (convergence or Accept) / **`phase-blocked`** (Stop) — this skill writes it |
| **standalone-dogfood** | same | inline | recent implementation context (spec, git diff, or described feature), running app | gate report + silent fixes | **none** — writes nothing to build state, needs no `.build-state.json` or spec dir. **Exception:** invoked by the orchestrator as a narrow-phase closure → writes `phase-complete`/`phase-blocked` |

**Inline, always.** It orchestrates leaf agents and subagents cannot spawn subagents (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md` Rule 1). Fix leaves inherit the session model. Every leaf brief ends with the containment string from that file. Agents never commit; **this skill commits.** A leaf's "surface to user / stop" rule becomes `status: blocked|needs-decision` plus a diagnosis — main decides.

**Image hygiene (Rule 9).** Main **never holds base64.** Every browse leaf saves screenshots to `/tmp/` and returns paths plus a text verdict; main triages on text and opens one screenshot only when a verdict is genuinely ambiguous.

Brain: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md`, `$AGENT=review`.

## Mode detection

- Arg `pipeline-review` → the phase gate. Also the resolution when `/build` routes here at `backend-complete` — **unless** `.build-state.json` carries `phaseCeremony: "narrow"`, which routes to standalone-dogfood instead; a narrow phase has no `validation.md` for pipeline-review to check against.
- Arg `standalone-dogfood`, a feature description, or **no arg with implementation clearly just finished outside `/build`** ("done", "just built X", a recent diff with no active phase) → standalone. **Also standalone** when the orchestrator explicitly invokes it to close a narrow phase — an explicit instruction, never inferred from the state file's presence.
- Ambiguous → ask once: review this build phase, or dogfood just what was built?

Standalone is **first-class**, not a fallback. "Tests pass" is not "the feature works."

## Briefing the two agents

**`dogfood`.** Pass: the optional persona line, the **one-sentence user goal** (pipeline: the app-purpose sentence from `outcome-card.md`; standalone: the *Original problem* sentence), the optional memorable-thing line from `design-brief.md ## Design intent`, the URL, credentials, a one-line **scope fence**, then for its Phase 2 the phase's stories, `validation.md` checks, and outcome-card.

**Never brief `dogfood`:** full `requirements.md`, `plan.md`, `handover.md`, the diff, what was built, the implementation conversation, or any hint about *how* anything works. Its blindness discipline depends on the caller withholding this.

**Scope fence — always.** Hand the agent ONE plain line: the capability live this phase (a door that opens) and the surfaces that are intentional placeholder scaffolding wired later (doors painted on). Derive it from `outcome-card.md` primary outcomes plus `product.md`'s App Map — never from `requirements.md`, `plan.md`, or the diff. A phase with no live task flow (a Phase-0 shell) gets: every screen is intentional static preview — judge whether screens render, navigate, and read clearly, not whether tasks complete.

**Reading the return.** `dogfood` already reports the three-signal gate verdict (functional / problem-resolved / no severe friction) plus severity-tagged findings. Triage them; never re-derive the gate. A finding on a surface this phase's card does not deliver is `expected-not-built`: LOW, informational, never a fix-trigger, regardless of the severity the agent proposed.

**Fresh instance per iteration.** Once an agent has seen a discrepancy it is no longer a clean read — re-verify always spawns a fresh instance (Rule 8).

---

## Mode: pipeline-review

Read `specs/YYYY-MM-DD-[feature]/validation.md` first. Missing → stop: there is no validation spec, `/build:spec` must run this phase first.

**Sequence:** Step 0 scope pre-check → Step 1 code review → Step 2 dogfood → fix loop → dogfood handoff → terminal write. Step 2 is one continuous session, empty → populated, **no DB resets between sub-passes**. Dead-ends and orphan endpoints were caught at spec time; never re-derive them.

**Scope.** `ui: false` → skip Step 2. Phase 0 → the walk explores the whole app, no shell regression. Phase 1+ → shell regression on top of the story walk.

### Step 0 — Scope drift pre-check

Read `plan.md` task groups, run `git diff main..HEAD --name-only`, and classify each group **DONE / PARTIAL / NOT DONE / SCOPE CREEP** in a one-line table. **Gate:** any group covering a primary user-flow story that is NOT DONE or PARTIAL → enter the fix loop with the incomplete groups as the brief, and do not open the browser this iteration. Non-primary NOT DONE → warn only.

### Step 1 — Code review, no browser

Write `currentSubStep: "review.code"`. **The browser does not open until the checks are green.**

**1a — Automated checks.** Run every check in `validation.md` (tsc, unit tests, one curl per API contract). Record exit code and `✓`/`✗`. Start the dev server if needed, poll ≤30s, never ask the user. Any `✗` is HIGH. Fast-fail before spending Opus on 1b.

**1b — `code-reviewer` agent.** Dispatch with target `main..HEAD` and the spec paths for context. It returns tagged, severity-scored findings. An empty or all-LOW return is only valid if the agent named what it actually traced.

**Triage.** HIGH and MEDIUM → fix loop. LOW → fix silently if trivial, else `T-N [polish]` to backlog. A `delete:`/`stdlib:`/`native:`/`yagni:`/`shrink:` finding crossing an API-contract or public-interface boundary needs a design decision — log the skip with a one-line reason, never auto-apply.

**1d — Re-verify.** Any fix from 1b changes code with no behavioral gate of its own, so re-run 1a. Any new `✗` is HIGH.

**Gate:** Step 2 opens only when 1a and 1d are green. 1b's surviving must-fix findings merge into the **same** fix loop as dogfood findings — never a second loop.

### Step 2 — Dogfood (skip entirely if `ui: false`)

Write `currentSubStep: "review.dogfood"`. **Browser-optional ladder:** an outcome already proven at Step 1's automated rung is not re-driven in the browser. The walk verifies only the last mile a test cannot — that the UI is *wired* to the behavior and *visibly confirms* success. A pure data or logic outcome with no new UI needs no browser.

**Dispatch `dogfood`** with the app-purpose sentence, URL, credentials, the scope fence, and — for its Phase 2 — every user story in **this** phase's `requirements.md` (never re-walk prior phases), every manual check in `validation.md`, and this phase's outcome-card. Phase 1+ also asks it to name the screens added this phase, confirm they are reachable from existing nav, and confirm return-to-main-app works.

**`final-phase`** (last roadmap phase — the orchestrator passes this arg). Run the walk **unfenced**: drop the scope fence and walk every Named Flow in `mission.md ## Master User Journey` plus every primary outcome from every `specs/*/outcome-card.md` on disk, oldest phase first. `expected-not-built` no longer applies; the whole product is in scope. This is the only cross-phase integration check in the stack, so a finding that "isn't this phase's problem" is exactly what it exists to catch. Same fix loop, same cap.

**Shell regression (Phase 1+, hardcoded, always runs regardless of `validation.md`):** (1) global nav renders and is interactive on a Phase-0 screen; (2) logo/home reaches the dashboard from a route added this phase; (3) auth still gates protected routes — unauthenticated redirects to login; (4) a toast fires on an action added this phase; (5) any nav item added this phase is in the right place and clickable. Fold these into the same dispatch as extra items to walk.

**Outcome-card grading.** Triage the findings, then grade each PRIMARY outcome from the story-walk results: the outcome verbatim, delivered recognizably Yes/Partial/No, on-screen signal against the card's "Success looks like" line. `Partial` or `No` → **HIGH**. A memorable-thing line in `design-brief.md ## Design intent` is judged from the agent's Phase-1 answer — `Partial`/`No` → **MEDIUM**. A legacy phase with no card grades the goal from `requirements.md`. **Grade caller-facing promises by what a caller can actually obtain, not by what the code names internally.**

---

## Mode: standalone-dogfood

The verification gate for implementation outside `/build`. Uses the app via `/browse`, never by reading code. Reads no state file, needs no spec dir, writes nothing to build state. **Exception:** invoked by the orchestrator to close a narrow phase, steps 0-6 run identically but a passing gate writes `phase-complete` (or `phase-blocked` on Stop). Never report-only.

0. **Automated checks and code review first** — a narrow-phase closure must not be weaker than pipeline-review. Typecheck, unit tests, one curl per API contract, green before the browser opens; then dispatch `code-reviewer` on `main..HEAD` and feed its findings into the same fix loop. On an ad-hoc run with no spec dir, derive the checks from what changed and scope the reviewer to the diff.
1. **Find the running app.** `lsof -i :3000 -i :3001 -i :5173 -i :8080 -i :4000 -i :8000 2>/dev/null | grep LISTEN` → use that port. None → read `package.json` `scripts.dev`/`scripts.start` or `CLAUDE.md`, start it in the background, wait ≤15s. Still nothing → ask once for the URL.
2. **Two distinct sentences.** From, in order: the arg, `specs/*/requirements.md`, `git diff HEAD~1 --stat` plus `git log -1 --format=%B`, else ask once. Write an **Original problem** (what the user could not do or worked around) and a **What was built**. If the two read the same you are verifying the implementation, not the problem — re-derive.
3. **Derive 2–4 scenarios from the problem, not the implementation.** A bug fix: scenario 1 reproduces the trigger and verifies it is gone. A new feature: scenario 1 is "user arrives with the problem and tries to solve it naturally", scenario 2 one edge case. Classify each **Simple** (navigate → verify) or **Flow** (action → verify → action → verify), documented before executing.
4. **Dispatch `dogfood`** with the classified scenarios as its Phase 2 walk and the *Original problem* sentence as its Phase 1 goal — nothing else, per the blind rule. Phase 1 runs first; Phase 2's findings should be new gaps, not a restatement.
5. **Three-signal gate.** The agent returns it. Do not output "done" until all three clear, even if the user said "finish up." Signal #2 fail → HIGH → fix loop. **Signal #3 only**, with 1 and 2 passed → surface a user decision: functionally complete, agent flagged [list], fix now or accept and move on?
6. **Fix loop**, then a Sonnet leaf writes the closing report, text only: `# Dogfood — [feature one-liner]` · `Gate: PASS/FAIL (#1/#2/#3)` · severity-tagged findings · one-line verdict. Forward it to the user.

---

## Fix loop (both modes)

**Every HIGH and MEDIUM** from any round triggers the loop. In pipeline-review the user sees nothing until the cap-hit binary — **silent auto-fix.** LOW: fix silently if trivial, else append to `backlog.md ## Dogfood polish` as `- [T-N] [polish] YYYY-MM-DD [Phase N] [description] — open`, threading dogfood bugs as `DF-N`. Never log LOWs in the report.

**A felt-impact fork is never silent.** Silent auto-fix is only for a finding with one correct outcome. Where the fix forces a felt choice — a slow list fixed by pagination versus caching — surface it as a fork (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`), then apply the pick. Legitimate even mid-loop.

**Iteration counter (pipeline only):** read `reviewIteration` (default 0), increment and write it back before each dispatch, preserving the other fields. Standalone tracks iteration in-conversation.

1. Collect every failure verbatim into a fix brief — failing checks with output, failing story records, outcome-card gaps — plus the verify command for each.
2. **Dispatch.** *pipeline-review:* wave-dispatched fix leaves, non-overlapping file sets, parallel (Rule 7) — Opus for structural and logic work, Sonnet for token, markup, and copy. Each brief carries its failures verbatim, never summarized; the spec paths and the design source; the verify commands; **verify-script red first, implement, green after — no fix without a passing verify script**; **root cause only, no refactoring, no scope or spec change; >3 hypotheses → return the diagnosis**; every file it touched, for the regression band; and the containment string. A felt-impact fork returns `status: needs-decision` plus the fork, never a silent pick. *standalone:* fix inline on main, root cause only, **commit each fix atomically** (`fix: [one-line]`) before re-verifying.
3. Read the summaries. Any `needs-decision` → surface the fork now, re-dispatch a fresh leaf with the pick. Union the touched files for the regression band and resolve cross-agent interface mismatches inline.
4. **Targeted re-verify, never a full re-round.** A failed automated check re-runs inline. A failed story or outcome item spawns parallel **fresh** Sonnet re-verify leaves, one per failed item — brief: the item's expected outcome and the URL; confirm this specific item works; screenshot to `/tmp/`, return paths and a fixed/still-failing verdict. **Thin regression band:** also re-run any story whose handler shares a file with the touched list, in the same batch. **Nothing else re-runs. The full dogfood session never repeats.**
5. Clean → terminal state, then the handoff, then the report. Still failing → increment and re-enter.

**Cap: 3.** `reviewIteration >= 3` (or 3 attempts per failing signal, standalone) with failures still present → surface **once** a binary, never a menu: the phase number, the count still failing after 3 attempts, a one-line list of each, and **Accept anyway** (complete with known issues) or **Stop**. Wait; no further attempts unless the user asks.

Accept → `step: "phase-complete"`. Stop → `step: "phase-blocked"`. Clean convergence → `phase-complete`. All three also write `reviewIteration: 0` and null `currentSubStep`.

---

## Dogfood handoff

Runs as the body of `phase-complete`, before any "phase complete" message. Skip it entirely only on `phase-blocked`.

**Idempotent and re-entrant.** `dogfoodPid` non-null and `kill -0 <pid>` succeeds → print a one-line reminder of URL, credentials, and bullets; no duplicate server, but **still run step 1** (commit and push are idempotent). Null or dead → run the full sequence. This server is separate from any one-shot test server Step 1 used.

**1 — Commit and push, always first.** Verify the branch is `phase-N-<slug>`; on `main` or the wrong branch, stop and surface. Stage exactly the `git status --porcelain` files — never `git add -A`. Commit `phase N complete: <summary>` with a HEREDOC and the `Co-Authored-By:` trailer. A suspicious staged set (`.env`, credentials, binaries, files outside `app/` and `specs/<phase>/`) → pause and ask. `git push -u origin <branch>`; rejected → do not force, fetch and surface; no `origin` → skip and say so once.

**2 — Pick the run command.** `package.json` dev → start → serve, via the lockfile's package manager. No dev script → skip the handoff and convey phase complete with no dev script.

**3 — Detect the LAN-reachable URL, Tailscale first.** (1) `tailscale ip -4 2>/dev/null | head -n 1` — the most reliable inter-device IP when the operator owns this device on the tailnet. (2) Fallback to the first non-loopback IPv4: `hostname -I | awk '{print $1}'`. (3) Port from the dev script, default `3000`. Final URL `http://<IP>:<PORT>`.

**4 — Seed credentials, only if the project has env-credential bootstrap.** Detect by grepping for `process.env.EMAIL`, `process.env.PASSWORD_HASH`, or a seed-user pattern; none → skip and omit the credentials line. If found: **password `dogfood`** (easy to type on a phone), bcrypt-hashed at the project's salt rounds (`bun -e 'import("bcrypt").then(b=>b.default.hash("dogfood",10).then(h=>console.log(h)))'` or the `node -e` equivalent); email `<project-basename>@<project-basename>.local`; persist to **`.env.dogfood`** at the project root (gitignored, reused on future phases); DB path `~/.config/<project-basename>/build-dogfood.db` on the project's DB-path env var.

**5 — Start the server, capture the PID.** `env $(cat .env.dogfood 2>/dev/null | xargs) <run-command> > /tmp/<project>-dogfood.log 2>&1 &` so `.env.dogfood` overrides the DB path and seed credentials. Write the PID to `dogfoodPid`; poll the URL ≤30s until 2xx or 3xx.

**6 — "What you can test" bullets.** The bullets ARE the card: one per PRIMARY outcome, its "Success looks like" signal as the what-should-happen half. A legacy phase with no card falls back to `requirements.md` stories. Operator voice per bullet: the screen or feature name, one sentence on what to do, one on what should happen. Primary stories only.

**7 — Print in operator voice.**
```
From your Mac (or phone), open:
  <URL>/<entry-route, default "/">
  Email: <email>
  Password: dogfood

---
What you can test
- <bullet 1>
- <bullet 2>
```
Close by conveying that the user can say "stop dogfood" to kill the server or leave it running. The `phase-complete` go/no-go gate is the orchestrator's; this skill holds no AUQ here.

**Stop or stale PID.** "stop dogfood" → `kill <pid>`, verify with `kill -0`, escalate to `kill -9` after 2s if alive, null the field. Already null → nothing is running; never hunt processes by name. On resume with a non-null `dogfoodPid`, verify `kill -0` and null it silently if dead.

**Dogfood feedback.** Each thing the user raises after handoff files as `DF-N` (`open`, their words as `R1`, tell them the ID), gets a root-cause fix, and threads follow-ups as `R2`/`R3`. On resolution: fixed → `fixed`; real-but-later → `T-N [side]` plus `deferred`; needs feature work → `T-N [roll-in]`. Refer by ID.

---

## Report

Written by main after a clean re-verify, or on Accept. Title `### Review Report — Phase [N]`, then date and iteration count ("0 — no issues found" if clean), then:

- **Outcome Card Verdict** — table: primary outcome / delivered? / signal seen against promised.
- **Code review** — checks N/N; agent findings N HIGH / N MEDIUM / none; M simplifications applied (−K lines).
- **Dogfood — Blind first impression** — what the blind pass found, or that the goal was reached cleanly.
- **Dogfood — Story and shell coverage** — story table; screen presence N/N; shell regression PASS or failures.
- **Auto-fixes Applied** — one line per fix, or "None."
- **Phase Verdict** — 1–2 sentences: did the phase deliver its card contract?

standalone uses the shorter closing report from its step 6 instead.

## Ground rules

1. **Judgment lives in the agents.** Never restate their catalogs here; supply scope and inputs, then gate on what they return.
2. **The browser never opens before the automated checks are green.**
3. **Silent auto-fix for HIGH and MEDIUM** — the user sees nothing until the cap-hit binary, except a felt-impact fork, which is never silent.
4. **Agents never commit; this skill commits.** Every leaf brief ends with the containment string from `subagent-policy.md`.
5. **Main never holds base64** — leaves save screenshots to `/tmp/` and return paths plus a text verdict.
6. **Never re-run the full dogfood session.** Re-verify is targeted, plus the thin regression band.
7. Never report "done" until the three-signal gate clears, even if the user says to finish up.
