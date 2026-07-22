---
name: build-review
description: >
  SDD review + dogfood skill (v3), two modes by explicit arg. **pipeline-review** (the /build
  phase gate): Round 1 code review — automated checks (tsc, unit tests, one curl per API contract),
  then the shared `code-reviewer` agent (all architecture/Fowler/ponytail lenses + the correctness
  bug-hunt, one pass) — re-verified, no browser; Round 2 dogfood through the shared `dogfood` agent
  (blind first-impression + guided story/shell/validation walk in one session), then
  outcome-card grading; every HIGH+MEDIUM finding
  auto-fixes silently in one wave-dispatched loop (cap 3 → Accept/Stop binary); then it owns the
  dogfood handoff and writes phase-complete | phase-blocked. **standalone-dogfood** (NON-pipeline):
  auto-fires whenever implementation finishes OUTSIDE /build — "done", "just built X", "just fixed
  X" — scoped to the just-built feature, runs the SAME two agents + fix loop, writes NOTHING to build
  state and needs no .build-state.json or spec dir. Trigger on
  /build-review, when /build reaches its review step, or when non-pipeline implementation just
  ended.
user-invocable: true
argument-hint: "pipeline-review | standalone-dogfood | [feature description to dogfood]"
---

> **Part of `/build`.** On a resume with an active `.build-state.json`, `pipeline-review` mode must enter through the `/build` orchestrator — it routes here; don't drive this skill off the state file directly (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`, which also lists the standalone-invocable modes — `standalone-dogfood` is one).

# /build-review — Code review and functional dogfood

One skill, two shared agents, two modes. **pipeline-review** is the `/build` phase gate; **standalone-dogfood** is the verification gate for any implementation that finishes outside the pipeline.

**The dedup this skill carries.** All review *judgment* — the lens catalog, the correctness bug-hunt, the blind-reviewer discipline, the guided-walk mechanics — lives in two agents, not here: `subagent_type: code-reviewer` (Opus) and `subagent_type: dogfood` (Sonnet). Both modes dispatch both agents; this skill owns only *orchestration* — what to pass them, how to gate on what they return, the fix loop, and the terminal writes. Historically this catalog was duplicated inline in both `build-review` and `build-lite-review`, which is exactly how they drifted; a shared agent definition removes that failure mode structurally. **The modes differ only in scope and terminal write:** pipeline adds shell regression + outcome-card grading and writes `phase-complete`/`phase-blocked`; standalone scopes to the just-built feature and writes nothing to build state — **except** when the `/build` orchestrator explicitly invokes standalone-dogfood to close out a narrow-ceremony phase (`phaseCeremony: "narrow"`), in which case it writes `phase-complete`/`phase-blocked` too (see Mode detection and standalone-dogfood below).

> **Judgment, not lookup.** The agents' severity tags keep calls *consistent* — they catch the obvious bug classes (hardcoded secrets, missing auth, broken pagination) without this skill needing a checklist. A quoted string here is intent to convey in plain language (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`), not a script, unless marked **exact**.

---

## Invocation contract

| Mode | Model | Mechanism | Inputs | Outputs | Terminal state |
|---|---|---|---|---|---|
| **pipeline-review** | Opus (main); `code-reviewer` + `dogfood` agents; Opus fix leaves | **inline** (in the `/build` session) | spec dir, `validation.md`, `outcome-card.md`, `requirements.md`, running app | review report + silent fixes + dogfood handoff | `phase-complete` (convergence or Accept) / `phase-blocked` (Stop) — **this skill writes it** |
| **standalone-dogfood** | Opus (main); `code-reviewer` + `dogfood` agents; Opus fix work | **inline** | recent implementation context (spec, git diff, or described feature), running app | gate report + silent fixes | **none** — writes nothing to build state, needs no `.build-state.json` or spec dir. **Exception:** invoked by the orchestrator as a narrow-phase closure → writes `phase-complete`/`phase-blocked` |

**Inline, always.** It orchestrates leaf agents (`code-reviewer`, `dogfood`, fix), and subagents can't spawn subagents — so it runs inline in the `/build` session and is never itself spawned as a subagent (one-level nesting, `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md` Rule 1); fix leaves inherit the session model.

**Leaf containment — every brief ends with this. exact:** *no spawning, no /build\*, no claude -p, no commit, no server, don't address the user — return content/paths only.* Agents never commit; **this skill commits.** Any "surface to user / stop" rule inside a leaf becomes `status: blocked|needs-decision` + diagnosis — main decides.

**Image hygiene (Rule 8).** Main **never holds base64** — every browse leaf saves screenshots to `/tmp/` and returns paths + a text verdict; main triages on text, opening one flagged screenshot only when a verdict is genuinely ambiguous.

---

## Mode detection

- Arg `pipeline-review` → the phase gate. Also the resolution when `/build` routes here at `backend-complete` — **unless** `.build-state.json` has `phaseCeremony: "narrow"` for the active phase, which routes to standalone-dogfood instead (below); a narrow phase never has a `validation.md` for pipeline-review to check against.
- Arg `standalone-dogfood`, or a feature description, or **no arg but implementation clearly just finished outside `/build`** ("done", "just built X", "just fixed X", a recent commit/diff with no active `.build-state.json` phase) → standalone. **Also standalone** when the orchestrator explicitly invokes it to close out an active phase with `phaseCeremony: "narrow"` — an explicit orchestrator instruction, not inferred from the state file's mere presence; in this one case, write `phase-complete`/`phase-blocked` on exit (see standalone-dogfood below).
- Genuinely ambiguous → ask once, plain language: review this build phase, or dogfood just what was built?

Standalone is **first-class**, not a fallback. It is the auto-verification gate for all non-pipeline work: "tests pass" ≠ "the feature works."

---

**Model split.** Main is Opus — it never does review or dogfood judgment itself, only orchestrates. `code-reviewer` (Opus) and `dogfood` (Sonnet) carry their own model pinned in their agent definition — this skill just dispatches them. Targeted re-verify and standalone report write-up stay Sonnet leaves. Scope pre-check, automated checks + re-verify, triage + outcome grading, terminal writes, dogfood handoff, and the cap-hit binary all run on main inline.

---

## Two shared agents — both modes dispatch both

`subagent_type: code-reviewer` (see its own definition for the full lens/Fowler/ponytail/correctness catalog and output format) covers Round 1's judgment; `subagent_type: dogfood` (blind first-impression, then a spec-aware story walk, in one continuous session) covers Round 2's. Each mode supplies the scope and inputs; the agents own the catalog and the discipline.

**Briefing `dogfood`.** Pass: the optional persona line, the **one-sentence user goal** (pipeline: the app-purpose sentence from `outcome-card.md`; standalone: the *Original problem* sentence), the optional memorable-thing line from `design-brief.md ## Design intent`, the URL, login credentials, a one-line **scope fence** (below), then (for its Phase 2) the phase's stories/`validation.md` checks/outcome-card. **Never brief:** `requirements.md` (full), `plan.md`, `handover.md`, the diff, what was built, the implementation conversation, or any hint about *how* anything works — the agent's own blindness discipline depends on the caller actually withholding this.

**Scope fence (always — stops the blind pass scoring not-yet-built surfaces as defects).** Main holds full context and hands the agent ONE plain line: the capability that is *live this phase* (a door that opens) and the surfaces that are *intentional placeholder scaffolding, wired in a later phase* (doors painted on). Derived from `outcome-card.md` primary outcomes + `product.md` App Map — never from `requirements.md`/`plan.md`/the diff. When this phase delivers no live task flow (a Phase-0 shell/preview), the fence reads: every screen is intentional static preview this phase — judge whether screens render, navigate, and read clearly, not whether tasks complete.

**Reading `dogfood`'s return.** It already reports the three-signal gate verdict (functional / problem-resolved / no severe friction) and a severity-tagged finding list — main triages those into the fix loop, it doesn't re-derive the gate. A finding on a surface this phase's card does not deliver → `expected-not-built`: LOW informational, never a fix-trigger, regardless of what severity the agent proposed.

**Fresh instance per iteration.** Once an agent has seen a discrepancy it is no longer a clean read — re-verify always spawns a fresh instance (subagent-policy Rule 7).

---

## pipeline-review

Input: `specs/YYYY-MM-DD-[feature]/validation.md`. Read it first; if missing, stop and convey there is no validation spec — `/build-spec` must run this phase first.

**Sequence:** Step 0 scope pre-check → Round 1 (code review, no browser) → Round 2 (dogfood) → fix loop → dogfood handoff → write terminal step. Round 2 is one continuous session (empty → populated, **no DB resets between sub-passes**) — a real user doesn't reset state. Dead-ends and orphan endpoints are caught at spec time (`/build-spec` reconciliation checks 7–8), not re-derived here.

### Round scope

`ui: false` → **skip Round 2 entirely, Round 1 only.** Phase 0 → the guided walk explores the whole app, no shell regression (the shell is being built this phase). Phase 1+ → shell regression runs (below) on top of the story walk.

### Step 0 — Scope drift pre-check

Read `plan.md` task groups; run `git diff main..HEAD --name-only`; classify each group **DONE / PARTIAL / NOT DONE / SCOPE CREEP** in a one-line table. **Gate:** any group covering a primary user-flow story that is NOT DONE or PARTIAL → enter the fix loop with the incomplete groups as the brief; do NOT open the browser this iteration. Non-primary NOT DONE → warn, don't trigger.

### Round 1 — Code review (no browser)

Write `currentSubStep: "review.code"`. The single code-quality gate — checks, an architecture critique, then a correctness bug-hunt. **The browser does not open until the checks are green.**

**1a — Automated checks.** Run every check in `validation.md` (tsc, unit tests, one curl per API contract). Record exit code + `✓`/`✗`. Start the dev server if needed, poll ≤30s, never ask the user. Any `✗` = HIGH. Fast-fail before spending Opus on 1b.

**1b — Code review (`code-reviewer` agent).** Dispatch `subagent_type: code-reviewer` with the target `main..HEAD` and the spec paths for context. It runs the full lens/Fowler/ponytail catalog and the correctness bug-hunt in one pass and returns tagged, severity-scored findings (see its own definition — this skill doesn't restate the catalog). Read the findings; an empty/all-LOW return is only valid if the agent named what it actually traced, per its own output contract.

**Triage the agent's return:** HIGH → fix loop. MEDIUM → fix loop. LOW → fix silently if trivial, else `T-N [polish]` to backlog. A `delete:`/`stdlib:`/`native:`/`yagni:`/`shrink:` finding that crosses an API-contract or public-interface boundary needs a design decision → log the skip with a one-line reason, don't auto-apply, regardless of the severity the agent proposed.

**1d — Re-verify.** Any fix applied from 1b's findings changes code with no behavioral gate of its own — re-run 1a's checks. Any new `✗` = HIGH.

**Gate:** Round 2 opens only when 1a + 1d are green. 1b's surviving must-fix findings merge into the **same** fix loop as dogfood findings — never a second loop.

### Round 2 — Dogfood (skip entirely if `ui: false`)

One continuous session inside the `dogfood` agent, empty → populated. Write `currentSubStep: "review.dogfood"`. **Browser-optional ladder — drive `/browse` only where it earns it:** an outcome already proven at Round 1's automated rung (tsc, unit/component tests, one curl per API contract) is not re-driven in the browser; the guided walk browser-verifies only the last mile a test can't — that the UI is *wired* to the behavior and *visibly confirms* success. A pure data/logic outcome with no new UI to exercise → assert it at the cheapest sufficient rung, no browser.

**Dispatch `subagent_type: dogfood`** with: the app-purpose sentence from `outcome-card.md`, URL, credentials, the scope fence (above), and — for its Phase 2 — every user story in **this** phase's `requirements.md` (never re-walk prior phases), every manual check in `validation.md`, and this phase's outcome-card. Phase 1+ also asks it to name the screens added this phase, confirm they're reachable from existing nav, and confirm return-to-main-app works.

**Shell regression (Phase 1+, hardcoded, on top of the agent's own story walk — always runs regardless of `validation.md`):** (1) global nav renders + interactive on a Phase-0 screen; (2) logo/home reaches the dashboard from a route added this phase; (3) auth still gates Phase-2 routes — unauthenticated → login redirect; (4) toast fires on a Phase-2 action; (5) any nav item added this phase is in the right place and clickable. Fold these into the same dispatch as extra items to walk.

**Reading the return + outcome-card grading (main, text).** The agent already returns the gate verdict and severity-tagged findings — triage those; any finding landing on a surface outside this phase's card is `expected-not-built` (LOW, informational) regardless of the severity the agent proposed. Then grade each PRIMARY outcome from the agent's story-walk results: the card outcome verbatim, delivered recognizably Yes/Partial/No, on-screen signal seen vs the card's "Success looks like" line. `Partial`/`No` on any primary outcome → **HIGH**. If `design-brief.md ## Design intent` has a memorable-thing line, judge whether the experience delivered that feeling from the agent's Phase-1 answer — `Partial`/`No` → **MEDIUM**. Legacy phase with no card → grade the user goal from `requirements.md`. **Grade caller-facing promises by what a caller can actually obtain, not by what the code names or caps internally.**

---

## standalone-dogfood

The verification gate for any implementation outside `/build`. Uses the app via `/browse`, never by reading code. **Reads no `.build-state.json`, requires no spec dir, writes nothing to build state** — organic/ad-hoc invocation ("done", "just built X"). **Exception:** when the orchestrator explicitly invokes this mode to close out a narrow-ceremony phase (`phaseCeremony: "narrow"`), steps 1-6 below run exactly the same, but on a passing gate write `.build-state.json` `step: "phase-complete"` (or `"phase-blocked"` on Stop) — the same terminal write pipeline-review owns, just from this mode. Never report-only.

1. **Find the running app.** `lsof -i :3000 -i :3001 -i :5173 -i :8080 -i :4000 -i :8000 2>/dev/null | grep LISTEN` → use that port. If none, read `package.json` `scripts.dev`/`scripts.start` or `CLAUDE.md`, start it in the background, wait ≤15s. Still nothing → ask once for the URL.
2. **Two distinct sentences.** From (in order) the arg, `specs/*/requirements.md` (most recent dated dir), `git diff HEAD~1 --stat` + `git log -1 --format=%B`, else ask once: write an **Original problem** (what the user couldn't do / kept hitting / worked around) and **What was built**. If the two read the same, you're verifying the implementation, not the problem — re-derive.
3. **Derive 2–4 scenarios from the problem, not the implementation.** Bug fix: scenario 1 reproduces the situation that triggered the bug and verifies it's gone. New feature: scenario 1 = "user arrives with the problem, tries to solve it naturally"; scenario 2 = one edge case (empty state, validation failure, error) if applicable. Classify each **Simple** (navigate → verify present/absent/correct) or **Flow** (action → verify intermediate → action → verify outcome); document each before executing.
4. **Dispatch `subagent_type: dogfood`** with the classified scenarios as its Phase 2 walk and the *Original problem* sentence as its Phase 1 goal (blind rule above — nothing else). Its Phase 1 runs first; Phase 2's findings should be new gaps, not restating Phase 1.
5. **Three-signal gate.** The agent already returns this — do not output "done"/"complete" until all three clear in its report, even if the user said "finish up." Signal #2 fail → HIGH → fix loop. **Signal #3 only (1+2 passed)** → surface a user decision: functionally complete, agent flagged [list], fix now or accept and move on?
6. **Fix loop**, then a Sonnet subagent writes the closing report (text-only): `# Dogfood — [feature one-liner]` · `Gate: PASS/FAIL (#1/#2/#3)` · severity-tagged findings · one-line verdict. Forward it to the user.

---

## Fix loop (shared shape)

**All HIGH and MEDIUM findings** from any round trigger the loop. In pipeline-review the user sees nothing until the cap-hit binary — **silent auto-fix.** LOW: fix silently if trivial; else append to `backlog.md ## Dogfood polish` as `- [T-N] [polish] YYYY-MM-DD [Phase N] [description] — open` (thread dogfood bugs as `DF-N`). Never log LOWs in the report.

**Felt-impact fork is never silent.** Silent auto-fix is for a finding with one correct outcome (one option strictly dominates). If a fix forces a choice the user will *feel* with no strictly-better option (a slow list fixed by pagination *vs.* caching), surface it as a **fork** — `AskUserQuestion`: the genuine options + each option's plain-language tradeoff + a recommended default — then apply the pick; legitimate even mid-loop. Record it in `docs/decisions.md ## User decisions` (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

**Iteration counter (pipeline only):** read `reviewIteration` from `.build-state.json` (default 0); increment and write back before each dispatch, preserving the other fields. Standalone tracks iteration in-conversation only.

**Loop body:**

1. Collect every failure verbatim into a fix brief — failing checks with output, failing story/check records, outcome-card gaps — plus the verify command for each.
2. **Dispatch** —
   - **pipeline-review:** wave-dispatched fix **leaves**, non-overlapping file sets, parallel (subagent-policy Rule 6) — Opus for structural/logic, Sonnet for token/markup/copy. Each brief carries: its failures verbatim (never summarized); spec paths (`requirements.md`, `plan.md`, `handover.md`); the verify commands; **`/code-harness` discipline — verify-script red first, implement, green after, no fix without a passing verify-script**; **root cause only — no refactoring, no scope/spec change; >3 hypotheses → return the diagnosis**; every file it touched (for the regression band); the containment line. **Felt-impact fork → `status: needs-decision`** + the fork, never a silent pick.
   - **standalone-dogfood:** fix inline on main, root cause only, **commit each fix atomically** (`fix: [one-line]`) before re-verifying.
3. Wait for all agents; read summaries (what fixed, what didn't, files touched). **Any `needs-decision`:** surface the fork now (`AskUserQuestion`), re-dispatch a fresh leaf with the pick. Union touched-files for the regression band; resolve cross-agent interface mismatches inline. (pipeline)
4. **Targeted re-verify — not a full re-round:**
   - Failed **automated check** (tsc/unit/curl): re-run the command inline.
   - Failed **story/outcome item**: spawn parallel **fresh** Sonnet re-verify subagents, one per failed item — brief: the item's expected outcome + URL; confirm this specific item works; screenshot; return fixed/still-failing + what was seen; save to `/tmp/`, paths only. Main reads text verdicts.
   - **Thin regression band:** also re-run any story whose handler shares a file with the touched-files list — include in the same parallel batch. **Nothing else re-runs. The full dogfood session never repeats.**
5. Clean → write terminal state + report. Still failing → increment, re-enter loop.

**Cap: 3.** `reviewIteration >= 3` (or 3 attempts per failing signal, standalone) with failures still present → surface **once** a binary (never a menu of fixes): the phase number, the count still failing after 3 attempts, a one-line list of each, and **Accept anyway** (mark complete with known issues) or **Stop**. Wait for the user; no further attempts unless they ask.

- pipeline **Accept** → `step: "phase-complete"`, `reviewIteration: 0`, null `currentSubStep`. **Stop** → `step: "phase-blocked"`, `reviewIteration: 0`, null `currentSubStep`.
- pipeline **clean convergence** → `step: "phase-complete"`, `reviewIteration: 0`, null `currentSubStep` — THEN the dogfood handoff, THEN the report.

---

## Dogfood handoff (pipeline-review only — this skill owns it)

Runs as the body of `phase-complete`, before any "phase complete" message. **Idempotent + re-entrant:** if `dogfoodPid` is non-null and `kill -0 <pid>` succeeds, print a one-line reminder of URL + credentials + bullets — do NOT start a duplicate server; **still run step 1** (commit+push is idempotent — the work may not be on origin yet). If `dogfoodPid` is null/dead, run the full sequence. Skip the whole handoff only if the gate is `phase-blocked`. The dogfood server is separate from any one-shot test server Round 1 used.

**1 — Commit + push (always first, idempotent).** Verify branch = `phase-N-<slug>` (on `main`/wrong → stop, surface). Stage exactly the `git status --porcelain` files (never `git add -A`), commit `phase N complete: <summary>` (HEREDOC + `Co-Authored-By:` trailer); a suspicious staged set (`.env`, credentials, binaries, files outside `app/` and `specs/<phase>/`) → pause and ask. `git push -u origin <branch>` (rejected → do NOT force, fetch + surface; no `origin` → skip, say so once).

**2 — Pick the run command.** `docs/deployment.md` if present; else `package.json` dev → start → serve via the lockfile's package manager. No dev script → skip the handoff, convey phase complete with no dev script.

**3 — Detect the LAN-reachable URL (Tailscale-first order).** (1) **Tailscale:** `tailscale ip -4 2>/dev/null | head -n 1` — most reliable inter-device IP when the operator owns this device on the tailnet (works Mac, phone, anywhere on the tailnet). (2) fallback first non-loopback IPv4: `hostname -I | awk '{print $1}'`. (3) port from the dev script, default `3000`. Final URL `http://<IP>:<PORT>`.

**4 — Seed dogfood credentials (only if the project has env-credential bootstrap).** Detect by grepping for `process.env.EMAIL`, `process.env.PASSWORD_HASH`, or a seed-user pattern; none → skip, omit the credentials line. If found: **password `dogfood`** (easy to type on a phone), bcrypt-hashed at the project's salt rounds (`bun -e 'import("bcrypt").then(b=>b.default.hash("dogfood",10).then(h=>console.log(h)))'` or `node -e` equivalent); email `<project-basename>@<project-basename>.local`; persist to **`.env.dogfood`** at project root (gitignored — reused on future phases); DB path `~/.config/<project-basename>/build-dogfood.db` on the project's DB-path env var.

**5 — Start the server, capture PID.** `env $(cat .env.dogfood 2>/dev/null | xargs) <run-command> > /tmp/<project>-dogfood.log 2>&1 &` — so `.env.dogfood` overrides DB path + seed creds. Write the PID to `.build-state.json` `dogfoodPid`; poll the URL ≤30s until 2xx/3xx.

**6 — "What you can test" bullets (from `outcome-card.md`).** The bullets ARE the card — one per PRIMARY outcome, using its "Success looks like" signal as the "what should happen" half (the user approved these exact promises at phase start; now they verify by hand). Legacy phase with no card → fall back to `requirements.md` stories. Per bullet, operator voice: lead with the screen/feature name, one short sentence on what to do, one short sentence on what should happen (the card's success line). One bullet per primary story only — secondary/negative-path stories are review territory, not dogfood.

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
- ...
```
Close by conveying the user can say "stop dogfood" to kill the server or leave it running; then the `phase-complete` go/no-go gate is the orchestrator's.

**Stop / stale PID.** "stop dogfood" (or equivalent): `kill <pid>`, verify `kill -0`, escalate to `kill -9` after 2s if alive, null the field. Null but asked to stop → nothing is running; do NOT hunt processes by name. On resume with a non-null `dogfoodPid`, verify `kill -0`; if dead, null it silently.

**Dogfood feedback (Reports loop).** Each thing the user raises after handoff → file as `DF-N` (`open`, their words as `R1`, tell them the ID), fix root-cause, thread follow-ups (`R2`/`R3`). On resolution: fixed → `fixed`; real-but-later → `T-N [side]` + `deferred`; needs feature work → `T-N [roll-in]`. Refer by ID.

---

## Report

Main writes it after a clean re-verify (or on Accept). Title `### Review Report — Phase [N]`; date + iteration count ("0 — no issues found" if clean); then these sections:

- **Outcome Card Verdict** — table: primary outcome / delivered? / signal seen vs promised (Partial/No = a gap, already auto-fixed before this renders).
- **Round 1 — Code review** — checks N/N; `code-reviewer` agent findings N HIGH / N MEDIUM / none, M simplifications applied (−K lines).
- **Dogfood — Blind first impression** — what the `dogfood` agent's blind pass found, or that the goal was reached cleanly.
- **Dogfood — Story & shell coverage** — story/check table; screen presence N/N; shell regression (Phase 1+) PASS or failures.
- **Auto-fixes Applied** — one line per fix, or "None."
- **Phase Verdict** — 1–2 sentences: did the phase deliver its card contract, ready for handoff?

(standalone uses the shorter closing report from its step 6 instead.)

---

## Brain integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md` with `$AGENT=review`, `$TAGS` from `tech-stack.md`. **Friction:** one entry per real bug — `Phase <N> review friction: <bug name>`; body = which sub-pass surfaced it, which story, how resolved, what check would catch it earlier. **Phase-wrap:** once after the final report — `Phase <N> review: <summary>`; body = what caught real bugs, what UX pattern to watch next.

---

## Ground rules

- **Two shared agents, two scopes.** Both modes dispatch `code-reviewer` and `dogfood` — no second copy of the lens catalog or the dogfood engine exists anywhere in this stack. Modes differ only in scope (pipeline adds shell regression + outcome grading) and terminal write (pipeline writes `phase-complete`/`phase-blocked`; standalone writes nothing — except a narrow-phase closure, which writes the same terminal state).
- **`/browse` only, inside the `dogfood` agent** — never Puppeteer, curl, or reading code to infer appearance. Screenshot every story outcome + meaningful screen at 1280px AND 390px. "Checked the code" is not UX verification.
- **Round 1 checks green before the browser (pipeline).** `code-reviewer` findings merge into the SAME fix loop as `dogfood` findings — never a second loop.
- **All HIGH + MEDIUM auto-fix silently (pipeline); a felt-impact fork is never a silent pick** — surface it. Agents never commit; this skill commits. Cap 3 → Accept/Stop binary, no further attempts unless the user asks.
- **Terminal state before any user-facing output (pipeline); standalone writes no build state.** User is non-technical — they see only the final report, the cap-hit binary, or a fork. Never ask which bug to prioritize or whether a deviation matters.
