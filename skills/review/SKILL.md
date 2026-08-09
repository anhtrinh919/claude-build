---
name: review
description: 'Code review and functional dogfood. **pipeline-review** is the /build phase gate: automated checks, the `code-reviewer` agent, then a `dogfood` coverage walk whose briefing goes to the user — code-review HIGH/MEDIUM and blocking dogfood findings are auto-fixed silently. **standalone-dogfood** fires when implementation ends outside /build.'
user-invocable: true
argument-hint: "pipeline-review | standalone-dogfood | [feature description to dogfood]"
---

# /build:review — code review and functional dogfood

Two modes over two shared agents. `code-reviewer` (Opus) owns the review judgment — the lens, Fowler, ponytail, and correctness catalog. `dogfood` (Sonnet) owns **coverage, not judgment**: it walks the app and returns a briefing of what it covered, what is broken, and what it never reached. The taste call is the user's, at the phase gate they already hold. This skill owns orchestration only — what to pass the agents, what to fix, the fix loop, the terminal writes.

Enter through `/build` for `pipeline-review` (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`). A quoted string is intent to convey in plain language (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`), never a script, unless marked **exact**.

## Invocation contract

| Mode | Model | Mechanism | Inputs | Outputs | Terminal step |
|---|---|---|---|---|---|
| **pipeline-review** | Opus main; `code-reviewer` + `dogfood` agents; Opus fix leaves | inline | spec dir, `validation.md`, `outcome-card.md`, `requirements.md`, `docs/rejected.md`, running app | review report + coverage briefing + silent fixes + dogfood handoff | **`phase-complete`** (convergence or Accept) / **`phase-blocked`** (Stop) — this skill writes it |
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

**`dogfood`.** Pass: the mode (`brief` or `gate`), the optional persona line, the **one-sentence user goal** (pipeline: the app-purpose sentence from `outcome-card.md`; standalone: the desired outcome, never the symptom), the URL, credentials, a one-line **scope fence**, then for its Phase 2 the explicit list of items to walk. **Every item is spelled out by the caller** — the agent derives nothing about what changed.

**Never brief `dogfood`:** full `requirements.md`, `plan.md`, `handover.md`, the diff, what was built, the implementation conversation, or any hint about *how* anything works. Its blindness discipline depends on the caller withholding this.

**Scope fence — always.** Hand the agent ONE plain line: the capability live this phase (a door that opens) and the surfaces that are intentional placeholder scaffolding wired later (doors painted on). Derive it from `outcome-card.md` primary outcomes plus `product.md`'s App Map — never from `requirements.md`, `plan.md`, or the diff. A phase with no live task flow (a Phase-0 shell) gets: every screen is intentional static preview — judge whether screens render, navigate, and read clearly, not whether tasks complete.

**Reading the return.** `dogfood` returns a **briefing, not a verdict**: a five-state coverage table (`ok` / `broken` / `unreachable` / `unsure` / `not-attempted`), a *Never reached* list, a *Blocking* list, its blind-arrival answers, and flat notes. Route it:

- **`broken`** — every one, whatever the item's importance → the fix loop. Never weigh whether a broken item "matters"; a `validation.md` check or a shell-regression item that fails is as blocking as a headline outcome.
- **The blocker behind an `unreachable` item** → the fix loop.
- **`unsure`** — settle it inline yourself; you hold the code, the diff, and Bash. **This is the only rule for `unsure`, and it decides the item outright.** Cleared → `ok`. Reveals a real defect → `broken`, into the loop. Still unsettled after one check → **`not observed`**: it does not block, it does not enter the loop, and it goes into the briefing by name. Never loop on an item with no reproducible failure.
- **`unreachable` and `not-attempted`** — into the briefing by name. **Never re-walked this phase**; the blocker is fixed, the items behind it stay unverified and the user is told so.
- **Notes and blind-arrival answers** — into the briefing verbatim. Never a fix-trigger, never re-graded into a severity.
- **`expected-not-built`** — informational, never a fix-trigger.

**Reconcile the count.** The coverage table must carry one row per item you briefed. Short → the missing items are `not-attempted`, and they go in the briefing; never assume a silent row passed.

**Never re-derive a *quality* verdict the agent did not give** — taste, friction, hierarchy, whether the thing felt good. Outcome-card delivery is a different thing and is yours to grade, from the coverage table only.

**Fresh instance per iteration.** Once an agent has seen a discrepancy it is no longer a clean read — re-verify always spawns a fresh instance (Rule 8).

---

## Mode: pipeline-review

Read `specs/YYYY-MM-DD-[feature]/validation.md` first. Missing → stop: there is no validation spec, `/build:spec` must run this phase first.

**Sequence:** Step 0 scope pre-check → Step 1 code review → Step 2 dogfood → fix loop → dogfood handoff → terminal write. Step 2 is one continuous session, empty → populated, **no DB resets between sub-passes**. Dead-ends and orphan endpoints were caught at spec time; never re-derive them.

**Scope.** `ui: false` → skip Step 2 — **except on `final-phase`**, where the unfenced walk runs regardless, because it is the stack's only cross-phase integration check and a flag on the last phase's spec must not delete it. **Unless the product has no browser surface at all**: no dev script, or every phase `ui: false`. Then skip it, say so in one line, and never enter a loop over Named Flows that no browser can reach. Phase 0 → the walk explores the whole app, no shell regression. Phase 1+ → shell regression on top of the story walk.

### Step 0 — Scope drift pre-check

Read `plan.md` task groups, run `git diff main..HEAD --name-only`, and classify each group **DONE / PARTIAL / NOT DONE / SCOPE CREEP** in a one-line table. **Gate:** any group covering a primary user-flow story that is NOT DONE or PARTIAL → enter the fix loop with the incomplete groups as the brief, and do not open the browser this iteration. Non-primary NOT DONE → warn only.

### Step 1 — Code review, no browser

Write `currentSubStep: "review.code"`. **The browser does not open until the checks are green.**

**1a — Automated checks.** Run every check in `validation.md` (tsc, unit tests, one curl per API contract). Record exit code and `✓`/`✗`. Start the dev server if needed, poll ≤30s, never ask the user. Any `✗` is HIGH. Fast-fail before spending Opus on 1b.

**1b — `code-reviewer` agent.** Dispatch with target `main..HEAD` and the spec paths for context. It returns tagged, severity-scored findings. An empty or all-LOW return is only valid if the agent named what it actually traced.

**Triage.** HIGH and MEDIUM → fix loop. LOW → fix silently if trivial, else `T-N [polish]` to backlog. A `delete:`/`stdlib:`/`native:`/`yagni:`/`shrink:` finding crossing an API-contract or public-interface boundary needs a design decision — log the skip with a one-line reason, never auto-apply.

**1d — Re-verify.** Any fix from 1b changes code with no behavioral gate of its own, so re-run 1a. Any new `✗` is HIGH.

**Gate:** Step 2 opens only when 1a and 1d are green. 1b's surviving must-fix findings merge into the **same** fix loop as dogfood findings — never a second loop.

### Step 2 — Dogfood (skip if `ui: false`, except on `final-phase` — see Scope)

Write `currentSubStep: "review.dogfood"`. **Browser-optional ladder:** an outcome already proven at Step 1's automated rung is not re-driven in the browser. The walk verifies only the last mile a test cannot — that the UI is *wired* to the behavior and *visibly confirms* success. A pure data or logic outcome with no new UI needs no browser.

**Dispatch `dogfood` in `brief` mode** with the app-purpose sentence, URL, credentials, the scope fence, and — for its Phase 2 — every user story in **this** phase's `requirements.md` (never re-walk prior phases), every manual check in `validation.md`, and this phase's outcome-card.

**Name every item yourself; the agent derives nothing.** Phase 1+ adds the new screens as walk items — *you* name each route, with "reachable from existing nav" and "returns to the main app" as its checks. Never ask the agent which screens are new: that fact does not exist in a running app, so the only way to answer is to read the repo, which its evidence rule forbids. **Any brief that can only be satisfied by opening a source file is a bug in this dispatch, not in the agent.**

**Source the names from this phase's `requirements.md` and `outcome-card.md`, never from `product.md`.** `product.md` is the phase-start anchor and its Screen Inventory still marks this phase's screens `planned` — replan flips them to `built`, and replan runs *after* review. Reading it here yields last phase's app.

**Whatever you name as a walk item is live for this walk.** The scope fence marks placeholder surfaces, and an item that appears in both is a contradiction the agent resolves as `expected-not-built` — silently declassifying every item you just added. Reconcile before dispatch: a named walk item never appears behind the fence.

**`final-phase`** (last roadmap phase — the orchestrator passes this arg). Run the walk **unfenced**: drop the scope fence and walk every Named Flow in `mission.md ## Master User Journey` plus every primary outcome from every `specs/*/outcome-card.md` on disk, oldest phase first. `expected-not-built` no longer applies; the whole product is in scope. This is the only cross-phase integration check in the stack, so a finding that "isn't this phase's problem" is exactly what it exists to catch. Same fix loop, same cap.

**Shell regression (Phase 1+, hardcoded, always runs regardless of `validation.md`).** Resolve every "added this phase" reference to a **named route, control, or nav label before dispatch** — the agent must never have to work out what is new: (1) global nav renders and is interactive on a Phase-0 screen; (2) logo/home reaches the dashboard from `<the named new route>`; (3) auth still gates protected routes — unauthenticated redirects to login; (4) a toast fires on `<the named new action>`; (5) `<the named new nav item>` is in the right place and clickable. Fold these into the same dispatch as extra items to walk.

**Outcome-card grading.** Grade each PRIMARY outcome from the coverage table: the outcome verbatim, delivered Yes/Partial/No/**not observed**, on-screen signal against the card's "Success looks like" line. `Partial` or `No` → **blocking** → fix loop. A legacy phase with no card grades the goal from `requirements.md`. **Grade caller-facing promises by what a caller can actually obtain, not by what the code names internally.** The memorable-thing line in `design-brief.md ## Design intent` is **not graded here** — the agent no longer judges taste, and the user judges it at the gate.

**An outcome grades `not observed` when *any* of its items is `unreachable`, `not-attempted`, or an unsettled `unsure` and none is `broken`.** A mixed outcome — some items verified, some never seen — is `not observed`, never `Partial`; `Partial` means observed and short, and would send an unverifiable outcome into a loop that can never clear it. Any `broken` item makes the outcome `No` on that item alone. Its blocker is blocking and gets fixed; the outcome itself does **not** block and does **not** re-enter the loop once the blocker clears, because nothing re-walks it. It carries into the briefing and the report by name, with an empty signal cell — never a signal no screenshot supports. `not observed` is a third thing: not delivered, not failed, not checked.

**Persist the briefing before leaving Step 2.** Write the coverage table, the never-reached list, and the notes to `specs/<phase>/dogfood-briefing.md`. The handoff and the report read that file, never the in-context return — a cold resume at `phase-complete` has no in-context return, and would otherwise print the handoff with the never-reached list silently missing.

**Refresh it at the end of every fix round**, after re-verify: flip each re-verified item to `ok` and drop it from the never-reached list. Grading and the report read the refreshed file, so a fixed outcome never grades `No` off a stale row, and the report never prints "3 broken" beside "3 auto-fixes applied".

**Absent file.** Step 2 does not always run — `ui: false`, a Step 0 scope-drift cap-out, a Step 1 check cap-out. Any reader that finds no file says exactly that: **"No coverage walk ran this phase — nothing was checked in a browser."** That is the honest line, and it is never silently omitted.

**On `final-phase`, every `broken` item is blocking on its own**, with no card lookup. A Named Flow from `mission.md` belongs to no outcome card, and grading only card outcomes would let the exact cross-phase break this walk exists to catch land in the notes.

---

## Mode: standalone-dogfood

The verification gate for implementation outside `/build` — the one mode that keeps a verdict, because no human gate follows it. Uses the app via `/browse`, never by reading code. Reads no state file, needs no spec dir, writes nothing to build state. **Exception:** invoked by the orchestrator to close a narrow phase, steps 0-6 run identically but a passing gate writes `phase-complete` (or `phase-blocked` on Stop). Never report-only.

0. **Automated checks and code review first** — a narrow-phase closure must not be weaker than pipeline-review. Typecheck, unit tests, one curl per API contract, green before the browser opens; then dispatch `code-reviewer` on `main..HEAD` and feed its findings into the same fix loop. On an ad-hoc run with no spec dir, derive the checks from what changed and scope the reviewer to the diff.
1. **Find the running app.** `lsof -i :3000 -i :3001 -i :5173 -i :8080 -i :4000 -i :8000 2>/dev/null | grep LISTEN` → use that port. None → read `package.json` `scripts.dev`/`scripts.start` or `CLAUDE.md`, start it in the background, wait ≤15s. Still nothing → ask once for the URL.
2. **Two distinct sentences.** From, in order: the arg, `specs/*/requirements.md`, `git diff HEAD~1 --stat` plus `git log -1 --format=%B`, else ask once. Write an **Original problem** (what the user could not do or worked around) and a **What was built**. If the two read the same you are verifying the implementation, not the problem — re-derive.
3. **Derive 2–4 scenarios from the problem, not the implementation.** A bug fix: scenario 1 reproduces the trigger and verifies it is gone. A new feature: scenario 1 is "user arrives with the problem and tries to solve it naturally", scenario 2 one edge case. Classify each **Simple** (navigate → verify) or **Flow** (action → verify → action → verify), documented before executing.
4. **Dispatch `dogfood` in `gate` mode** with the classified scenarios as its Phase 2 walk and the *Original problem* sentence as its Phase 1 goal — nothing else, per the blind rule. Phase 1 runs first; Phase 2's findings should be new gaps, not a restatement.
5. **Primary-flow gate — this mode only.** Standalone often has no human looking right behind it, so unlike pipeline-review it keeps a verdict. The agent returns one line on the stated goal: **Yes / No / unsure**. `Yes` passes. **`No` → blocking → fix loop.** **`unsure` is not a pass** — settle it inline yourself, and if it is still unsettled, treat it as `No`; this is the mode with no human gate behind it, so an unverified primary goal must never read as done. Never report "done" while the goal is `No` or unsettled, even if the user said "finish up." Everything below the goal — notes, `unreachable`, `not-attempted` — is briefing, exactly as in pipeline-review.
6. **Fix loop**, then a Sonnet leaf writes the closing report, text only: `# Dogfood — [feature one-liner]` · `Primary goal: reached / not reached` · coverage counts · never-reached list · notes · one-line verdict. Forward it to the user.

---

## Fix loop (both modes)

**What triggers the loop, by source:**

- **`code-reviewer`** — every HIGH and MEDIUM, unchanged. Code review keeps its full severity ladder.
- **Automated checks** — every `✗`.
- **`dogfood`** — **every `broken` item**, plus the blocker behind an `unreachable` one. No weighing: a failed shell-regression item or `validation.md` check enters the loop exactly like a headline outcome. Nothing else does — notes, blind-arrival answers, and `unsure` items the settle check cleared go to the user in the briefing, which is the point of handing them a briefing.

In pipeline-review the user sees nothing until the cap-hit binary — **silent auto-fix.** A non-blocking dogfood note worth keeping goes to `backlog.md ## Dogfood polish` as `- [T-N] [polish] YYYY-MM-DD [Phase N] [description] — open`, threading dogfood bugs as `DF-N`; the rest ride in the briefing only.

**A felt-impact fork is never silent.** Silent auto-fix is only for a finding with one correct outcome. Where the fix forces a felt choice — a slow list fixed by pagination versus caching — surface it as a fork (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`), then apply the pick. Legitimate even mid-loop.

**Iteration counter (pipeline only):** read `reviewIteration` (default 0), increment and write it back before each dispatch, preserving the other fields. Standalone tracks iteration in-conversation.

1. Collect every failure verbatim into a fix brief — failing checks with output, failing story records, outcome-card gaps — plus the verify command for each.
2. **Dispatch.** *pipeline-review:* wave-dispatched fix leaves, non-overlapping file sets, parallel (Rule 7) — Opus for structural and logic work, Sonnet for token, markup, and copy. Each brief carries its failures verbatim, never summarized; the spec paths and the design source; the verify commands; **verify-script red first, implement, green after — no fix without a passing verify script**; **root cause only, no refactoring, no scope or spec change; >3 hypotheses → return the diagnosis**; every file it touched, for the regression band; and the containment string. A felt-impact fork returns `status: needs-decision` plus the fork, never a silent pick. *standalone:* fix inline on main, root cause only, **commit each fix atomically** (`fix: [one-line]`) before re-verifying.
3. Read the summaries. Any `needs-decision` → surface the fork now, re-dispatch a fresh leaf with the pick. Union the touched files for the regression band and resolve cross-agent interface mismatches inline.
4. **Targeted re-verify, never a full re-round.** A failed automated check re-runs inline. A failed story or outcome item spawns parallel **fresh** Sonnet re-verify leaves, one per failed item — brief: the item's expected outcome and the URL; confirm this specific item works; screenshot to `/tmp/`, return paths and a fixed/still-failing verdict. **Thin regression band:** also re-run any story whose handler shares a file with the touched list, in the same batch. **Nothing else re-runs. The full dogfood session never repeats.**
5. Clean → terminal state, then the handoff, then the report. Still failing → increment and re-enter.

**Cap: 3.** `reviewIteration >= 3` (or 3 attempts per failing signal, standalone) with failures still present → surface **once** a binary, never a menu: the phase number, the count still failing after 3 attempts, a one-line list of each, and **Accept anyway** (complete with known issues) or **Stop**. Wait; no further attempts unless the user asks.

**Terminal writes are pipeline-review's, plus the narrow-phase-closure exception — never an ad-hoc standalone run.** Accept → `step: "phase-complete"`. Stop → `step: "phase-blocked"`. Clean convergence → `phase-complete`. All three also write `reviewIteration: 0` and null `currentSubStep`. **The test is the orchestrator's explicit narrow-closure instruction, and nothing else** — never the presence of a `.build-state.json`. Without that instruction a standalone run **writes no state at all**: creating one invents a build project that does not exist, and overwriting an existing `step` destroys a live resume position mid-phase.

**Clean convergence means:** automated checks green, no surviving `code-reviewer` must-fix, and no blocking dogfood finding. It does **not** mean the dogfood agent approved of the app — it no longer approves of anything. Unreached items, `unsure` items, and notes do not block `phase-complete`; they must all reach the user in the handoff instead.

---

## Dogfood handoff

Runs as the body of `phase-complete`, before any "phase complete" message.

**On `phase-blocked`, run none of steps 1-7.** The phase failed: nothing is committed, nothing is pushed, no server starts, no bullets print. **Print one thing — the "Worth a look" list**, sourced exactly as step 6 sources it. `phase-blocked` never auto-resumes, so this is the last moment the user can learn what was never checked. Never let a blocked phase reach step 1; it would commit and push a failed phase under the message "phase N complete".

**Idempotent and re-entrant.** `dogfoodPid` non-null and `kill -0 <pid>` succeeds → print a one-line reminder of URL, credentials, and bullets; no duplicate server, but **still run step 1** (commit and push are idempotent). Null or dead → run the full sequence. This server is separate from any one-shot test server Step 1 used.

**1 — Commit and push, always first.** Verify the branch is `phase-N-<slug>`; on `main` or the wrong branch, stop and surface. Stage exactly the `git status --porcelain` files — never `git add -A`. Commit `phase N complete: <summary>` with a HEREDOC and the `Co-Authored-By:` trailer. A suspicious staged set (`.env`, credentials, binaries, files outside `app/` and `specs/<phase>/`) → pause and ask. `git push -u origin <branch>`; rejected → do not force, fetch and surface; no `origin` → skip and say so once.

**2 — Pick the run command.** `package.json` dev → start → serve, via the lockfile's package manager. No dev script → skip the handoff and convey phase complete with no dev script.

**3 — Detect the LAN-reachable URL, Tailscale first.** (1) `tailscale ip -4 2>/dev/null | head -n 1` — the most reliable inter-device IP when the operator owns this device on the tailnet. (2) Fallback to the first non-loopback IPv4: `hostname -I | awk '{print $1}'`. (3) Port from the dev script, default `3000`. Final URL `http://<IP>:<PORT>`.

**4 — Seed credentials, only if the project has env-credential bootstrap.** Detect by grepping for `process.env.EMAIL`, `process.env.PASSWORD_HASH`, or a seed-user pattern; none → skip and omit the credentials line. If found: **password `dogfood`** (easy to type on a phone), bcrypt-hashed at the project's salt rounds (`bun -e 'import("bcrypt").then(b=>b.default.hash("dogfood",10).then(h=>console.log(h)))'` or the `node -e` equivalent); email `<project-basename>@<project-basename>.local`; persist to **`.env.dogfood`** at the project root (gitignored, reused on future phases); DB path `~/.config/<project-basename>/build-dogfood.db` on the project's DB-path env var.

**5 — Start the server, capture the PID.** `env $(cat .env.dogfood 2>/dev/null | xargs) <run-command> > /tmp/<project>-dogfood.log 2>&1 &` so `.env.dogfood` overrides the DB path and seed credentials. Write the PID to `dogfoodPid`; poll the URL ≤30s until 2xx or 3xx.

**6 — "What you can test" bullets, plus what was not checked.** The bullets ARE the card: one per PRIMARY outcome, its "Success looks like" signal as the what-should-happen half. A legacy phase with no card falls back to `requirements.md` stories. Operator voice per bullet: the screen or feature name, one sentence on what to do, one on what should happen. Primary stories only.

**Mark any PRIMARY outcome graded `not observed` in its own bullet** — "not checked this round" in plain words, on the bullet itself. An unmarked bullet reads as verified, and the user cannot connect an outcome to a screen name buried in the list below.

**Then a second, shorter list — "Worth a look" — read from `specs/<phase>/dogfood-briefing.md`, not from memory:** every item the scout never reached and why, everything `not-attempted`, anything left unsettled, and its blind-arrival stalls. No file → the one honest line that no coverage walk ran. Plain words, no severities, no state names. Omit the heading only when all three are genuinely empty. **Never drop an item to keep the list short** — an unmentioned screen reads as a checked screen, and that is the failure this handoff exists to prevent.

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

Worth a look
- <not reached, and what blocked it>
- <couldn't tell either way>
```
Close by conveying that the user can say "stop dogfood" to kill the server or leave it running. The `phase-complete` go/no-go gate is the orchestrator's; this skill holds no AUQ here.

**Stop or stale PID.** "stop dogfood" → `kill <pid>`, verify with `kill -0`, escalate to `kill -9` after 2s if alive, null the field. Already null → nothing is running; never hunt processes by name. On resume with a non-null `dogfoodPid`, verify `kill -0` and null it silently if dead.

**Dogfood feedback.** Each thing the user raises after handoff files as `DF-N` (`open`, their words as `R1`, tell them the ID), gets a root-cause fix, and threads follow-ups as `R2`/`R3`. On resolution: fixed → `fixed`; real-but-later → `T-N [side]` plus `deferred`; needs feature work → `T-N [roll-in]`. Refer by ID.

---

## Report

Written by main after a clean re-verify, or on Accept. Title `### Review Report — Phase [N]`, then date and iteration count ("0 — no issues found" if clean), then:

- **Outcome Card Verdict** — table: primary outcome / delivered? / signal seen against promised.
- **Code review** — checks N/N; agent findings N HIGH / N MEDIUM / none; M simplifications applied (−K lines).
- **Dogfood — Coverage** — the state counts (N ok / N broken / N unreachable / N unsure / N not-attempted), summing to the number of items briefed; story table; screen presence N/N; shell regression PASS or failures. No walk ran → say so in one line instead.
- **Dogfood — Never reached** — every `unreachable` item with its blocker, every `not-attempted` item by name, and every outcome graded `not observed`. "None" if none. Never omitted.
- **Dogfood — Blind arrival and notes** — the scout's three answers and its flat notes, verbatim, ungraded.
- **Auto-fixes Applied** — one line per fix, or "None."
- **Phase Verdict** — 1–2 sentences: did the phase deliver its card contract?

standalone uses the shorter closing report from its step 6 instead.

## Ground rules

1. **Judgment lives in the agents.** Never restate their catalogs here; supply scope and inputs, then gate on what they return.
2. **The browser never opens before the automated checks are green.**
3. **Silent auto-fix for code-review HIGH/MEDIUM and blocking dogfood findings** — the user sees nothing until the cap-hit binary, except a felt-impact fork, which is never silent.
4. **Agents never commit; this skill commits.** Every leaf brief ends with the containment string from `subagent-policy.md`.
5. **Main never holds base64** — leaves save screenshots to `/tmp/` and return paths plus a text verdict.
6. **Never re-run the full dogfood session.** Re-verify is targeted, plus the thin regression band.
7. **`dogfood` reports coverage; the user judges quality.** Never re-derive a *quality* verdict — taste, friction, hierarchy, whether it felt good — never re-grade its notes into severities, and never let it read the repo. Outcome-card delivery is not a quality verdict: it is main's grade, made from the coverage table alone, and it is required.
8. **Never let a coverage gap go unsaid.** An unreached item, an unsettled `unsure`, or a cap-hit leftover the user is not told about reads to them as a checked item. Silence is the one failure this design cannot absorb.
9. In `standalone-dogfood` only, never report "done" while the primary-flow gate says the goal is not reachable, even if the user says to finish up.
