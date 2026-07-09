---
name: build-review
description: >
  SDD review + dogfood skill (v2), two modes by explicit arg. **pipeline-review** (the /build
  phase gate): Round 1 code review — automated checks (tsc, unit tests, one curl per API contract),
  an architecture pass (design lenses incl. honest-mechanism/anti-hack + reinvention/simplification,
  plus Fowler's baseline code smells),
  a correctness pass (concrete-failure-scenario bug hunt) — re-verified, no browser; Round 2 dogfood through the shared
  browser-review engine (blind first-impression + guided story/shell/validation walk), then
  outcome-card grading; every HIGH+MEDIUM finding
  auto-fixes silently in one wave-dispatched loop (cap 3 → Accept/Stop binary); then it owns the
  dogfood handoff and writes phase-complete | phase-blocked. **standalone-dogfood** (NON-pipeline):
  auto-fires whenever implementation finishes OUTSIDE /build — "done", "just built X", "just fixed
  X" — scoped to the just-built feature, runs the SAME engine + fix loop, writes NOTHING to build
  state and needs no .build-state.json or spec dir. Trigger on
  /build-review, when /build reaches its review step, or when non-pipeline implementation just
  ended.
user-invocable: true
argument-hint: "pipeline-review | standalone-dogfood | [feature description to dogfood]"
---

# /build-review — Code review and functional dogfood

One skill, one browser-review engine, two modes. **pipeline-review** is the `/build` phase gate; **standalone-dogfood** is the verification gate for any implementation that finishes outside the pipeline.

**The dedup this skill carries.** There is now exactly ONE browser-test engine: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/browser-review-engine.md`. Both modes call it. Historically the pipeline review ran its *own* dogfood session directly, duplicating the standalone engine — that duplication is gone. **The modes differ only in scope and terminal write:** pipeline adds shell regression + outcome-card grading and writes `phase-complete`/`phase-blocked`; standalone scopes to the just-built feature and writes nothing to build state — **except** when the `/build` orchestrator explicitly invokes standalone-dogfood to close out a narrow-ceremony phase (`phaseCeremony: "narrow"`), in which case it writes `phase-complete`/`phase-blocked` too (see Mode detection and standalone-dogfood below).

> **Judgment, not lookup.** The severity tags and lens names keep calls *consistent* — the model catches the obvious bug classes (hardcoded secrets, missing auth, broken pagination) without a checklist. A quoted string is intent to convey in plain language (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`), not a script, unless marked **exact**.

---

## Invocation contract

| Mode | Model | Mechanism | Inputs | Outputs | Terminal state |
|---|---|---|---|---|---|
| **pipeline-review** | Opus (main); Sonnet browse leaves; Opus adversarial + fix leaves | **inline** (in the `/build` session) | spec dir, `validation.md`, `outcome-card.md`, `requirements.md`, running app | review report + silent fixes + dogfood handoff | `phase-complete` (convergence or Accept) / `phase-blocked` (Stop) — **this skill writes it** |
| **standalone-dogfood** | Opus (main); Sonnet browse leaves; Opus fix work | **inline** | recent implementation context (spec, git diff, or described feature), running app | gate report + silent fixes | **none** — writes nothing to build state, needs no `.build-state.json` or spec dir. **Exception:** invoked by the orchestrator as a narrow-phase closure → writes `phase-complete`/`phase-blocked` |

**Inline, always.** It orchestrates leaf subagents (browse, adversarial, fix), and subagents can't spawn subagents — so it runs inline in the `/build` session and is never itself spawned as a subagent (one-level nesting, `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md` Rule 1); fix leaves inherit the session model.

**Leaf containment — every brief ends with this. exact:** *no spawning, no /build\*, no claude -p, no commit, no server, don't address the user — return content/paths only.* Agents never commit; **this skill commits.** Any "surface to user / stop" rule inside a leaf becomes `status: blocked|needs-decision` + diagnosis — main decides.

**Image hygiene (Rule 8).** Main **never holds base64** — every browse leaf saves screenshots to `/tmp/` and returns paths + a text verdict; main triages on text, opening one flagged screenshot only when a verdict is genuinely ambiguous.

---

## Mode detection

- Arg `pipeline-review` → the phase gate. Also the resolution when `/build` routes here at `backend-complete` — **unless** `.build-state.json` has `phaseCeremony: "narrow"` for the active phase, which routes to standalone-dogfood instead (below); a narrow phase never has a `validation.md` for pipeline-review to check against.
- Arg `standalone-dogfood`, or a feature description, or **no arg but implementation clearly just finished outside `/build`** ("done", "just built X", "just fixed X", a recent commit/diff with no active `.build-state.json` phase) → standalone. **Also standalone** when the orchestrator explicitly invokes it to close out an active phase with `phaseCeremony: "narrow"` — an explicit orchestrator instruction, not inferred from the state file's mere presence; in this one case, write `phase-complete`/`phase-blocked` on exit (see standalone-dogfood below).
- Genuinely ambiguous → ask once, plain language: review this build phase, or dogfood just what was built?

Standalone is **first-class**, not a fallback. It is the auto-verification gate for all non-pipeline work: "tests pass" ≠ "the feature works."

---

**Model split** (rule: authoring/deciding → Opus, executing → Sonnet). Main is Opus. **Sonnet leaves:** blind reviewer, scenario/guided walk, targeted re-verify, standalone report write-up. **Opus leaves:** the architecture pass, the correctness pass, and the fix leaves. Scope pre-check, automated checks + re-verify, triage + outcome grading, terminal writes, dogfood handoff, and the cap-hit binary all run on main inline.

---

## The one engine — both modes call it

Read `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/browser-review-engine.md` before either mode. It owns how leaves are briefed, what the gate checks, and how the loop iterates; each mode supplies the scope. Its four engines map onto this skill:

| Engine (in the shared file) | pipeline-review uses it as | standalone uses it as |
|---|---|---|
| 1 — Scenario subagent (one clean `/browse` pass per scenario → PASS/FAIL/UNEXPECTED/MISSING) | the **guided walk** (this phase's stories + shell regression + `validation.md` checks + screen presence) | the 2–4 derived scenarios |
| 2 — Blind naive reviewer (a subagent that *cannot* see the implementation) | the **blind first-impression sub-pass** | the naive-reviewer pass |
| 3 — Three-signal gate (functional + problem-resolved + no severe friction) | the browse gate (+ outcome-card grading on top) | the completion gate |
| 4 — Fix loop (root-cause fix, fresh re-verify subagents, cap 3) | fed into the wave-dispatched fix loop below | inline fix loop |

### Blind-reviewer hard rule (verbatim — do not leak)

The blind sub-pass is the whole point of the engine: *you built this and can't unsee it.* The reviewer must be blind to the solution to give a real first-impression read.

**Pass ONLY:** the optional persona line, the **one-sentence user goal** (pipeline: the app-purpose sentence from `outcome-card.md`; standalone: the *Original problem* sentence), the optional memorable-thing line from `design-brief.md ## Design intent`, the URL, login credentials, and — on a `feature`/`rebuild` phase — a one-line **scope fence** (below). **The caller MUST NOT brief:** `requirements.md` (full), `plan.md`, `handover.md`, the diff, what was built, the implementation conversation, or any hint about *how* anything works or *which* feature is being tested. Nothing else.

**Scope fence (feature/rebuild only — stops the blind pass scoring not-yet-built surfaces as defects).** Main holds full context and hands the reviewer ONE plain line: the capability that is *live this phase* (a door that opens) and the surfaces that are *intentional placeholder scaffolding, wired in a later phase* (doors painted on). Derived from `outcome-card.md` primary outcomes + `product.md` App Map — never from `requirements.md`/`plan.md`/the diff — so it names *what's on the exam*, not *the solution*. The reviewer judges only the live capability; a placeholder it lands on is noted "not built yet," never scored broken. On an `initial` phase the fence instead reads: every screen is intentional static preview this phase — judge whether screens render, navigate, and read clearly, not whether tasks complete. A scope boundary is not a solution leak — blind to *how*, not to *what's in scope*.

The reviewer answers (engine's template): goal Yes/Partial/No + the on-screen signal that told them so ("I'm not sure" if none); where they hesitated/guessed, cited to the exact screen; the anti-pattern scan (re-read text twice? didn't know what to click? felt broken? competing elements? success inferred not shown? developer language? mobile overflow at 390×844?).

### Three-signal gate (verbatim)

Not done until **all three** are true:

1. **Functional** — every scenario returns PASS, UNEXPECTED, or MISSING (no FAILs). `UNEXPECTED`/`MISSING` are informational (LOW unless escalated) — they do NOT trigger a fix iteration.
2. **Problem-resolved** — the blind reviewer reports "Yes" to the goal AND can point to specific on-screen feedback that told them so. If a memorable-thing line was passed, the answer to it is at least "yes, it felt that way."
3. **No severe friction** — no "I didn't know what to do" or "I had to infer success" on the primary path.

**Severity from blind findings:** goal not accomplishable → **HIGH**; "I'm not sure it worked" on the primary goal → **MEDIUM** min; "had to guess what to click" / "re-read text twice" / "no feedback after action" / memorable-thing miss → **MEDIUM**; other anti-pattern hits → **LOW** unless they materially blocked the goal. **A finding on a surface this phase's card does not deliver → `expected-not-built`: LOW informational, never a fix-trigger** — a placeholder that invites action then dead-ends is at most a LOW "label it coming-soon," not HIGH/MEDIUM.

**Fresh instance per iteration.** Once a leaf has seen a discrepancy it is no longer a clean read — re-verify always spawns a fresh subagent (subagent-policy Rule 7).

---

## pipeline-review

Input: `specs/YYYY-MM-DD-[feature]/validation.md`. Read it first; if missing, stop and convey there is no validation spec — `/build-spec` must run this phase first.

**Sequence:** Step 0 scope pre-check → Round 1 (code review, no browser) → Round 2 (dogfood) → fix loop → dogfood handoff → write terminal step. Round 2 is one continuous session (empty → populated, **no DB resets between sub-passes**) — a real user doesn't reset state. Dead-ends and orphan endpoints are caught at spec time (`/build-spec` reconciliation checks 7–8), not re-derived here.

### Phase-type detection

Read `requirements.md` frontmatter `type`. **Missing/unrecognized → default `initial`** (full coverage is safer than narrowing on ambiguity).

| `type` | Shell regression (guided walk) | Blind scope |
|---|---|---|
| `initial` | Skip | Explore the entire app |
| `feature` | Run | New screens + connection points (reachable from existing nav? can you return to the main app from inside?) |
| `rebuild` | Skip (shell redesigned this phase) | Explore the entire app |
| `ui: false` | — | **Skip Round 2 entirely — Round 1 only** |

### Step 0 — Scope drift pre-check

Read `plan.md` task groups; run `git diff main..HEAD --name-only`; classify each group **DONE / PARTIAL / NOT DONE / SCOPE CREEP** in a one-line table. **Gate:** any group covering a primary user-flow story that is NOT DONE or PARTIAL → enter the fix loop with the incomplete groups as the brief; do NOT open the browser this iteration. Non-primary NOT DONE → warn, don't trigger.

### Round 1 — Code review (no browser)

Write `currentSubStep: "review.code"`. The single code-quality gate — checks, an architecture critique, then a correctness bug-hunt. **The browser does not open until the checks are green.**

**1a — Automated checks.** Run every check in `validation.md` (tsc, unit tests, one curl per API contract). Record exit code + `✓`/`✗`. Start the dev server if needed, poll ≤30s, never ask the user. Any `✗` = HIGH. Fast-fail before spending Opus on 1b.

**1b — Architecture pass (Opus subagent).** Target = `main..HEAD`. Tests catch logic bugs; the harness catches contract bugs; nothing else catches **shape** bugs. Default stance adversarial — assume the shape is wrong until findings prove otherwise; generic praise is a failure mode. Apply every lens below — one pass now, since it reads the same whole diff (this also folds in the old ponytail-review simplify pass); skip a lens only when structurally inapplicable or already caught under another name; verify every finding against the actual code before returning it.

Each finding names a concrete revision (collapse / inline / merge / delete / consolidate / rename / move-upstream) — no vague "consider refactoring."

| Lens | One-line frame |
|---|---|
| Module depth | interface as complex as its implementation = shallow, net-negative → collapse into the only caller |
| Abstraction necessity | rule of three — 1–2 instances don't earn an abstraction → inline; defer until the 3rd lands |
| Data-flow legibility | tracing one request shouldn't bounce through more files than it has steps → inline the pass-through layer |
| Seam placement | boundaries belong around things that change together → merge what always changes in the same commit |
| Information hiding vs relabeling | a name that hides nothing is a rename → delete the wrapper, call the library directly |
| Logic consolidation | one rule, one place — duplicated rules drift → consolidate; extract the threshold to one constant |
| Naming honesty | name a noun the domain knows, not a `Manager`/`Handler`/`Util` role → rename; split the kitchen-sink `Util` |
| Reinvention & dead weight (ponytail) | reimplemented stdlib/native, a new dep for what a few lines do, speculative flexibility with no 2nd caller, dead abstraction → replace with the stdlib/native/platform primitive, delete the speculative code; the whole-diff view also catches duplication *across* parallel build agents — consolidate to one; report net lines removed |
| Mysterious Name (Fowler) | a function, variable, or type whose name doesn't reveal what it does or holds → rename it; if no honest name comes, the design's murky |
| Duplicated Code (Fowler) | the same logic shape appears in more than one hunk or file in the change → extract the shared shape, call it from both |
| Feature Envy (Fowler) | a method that reaches into another object's data more than its own → move the method onto the data it envies |
| Data Clumps (Fowler) | the same few fields or params keep travelling together (a type wanting to be born) → bundle them into one type, pass that |
| Primitive Obsession (Fowler) | a primitive or string standing in for a domain concept that deserves its own type → give the concept its own small type |
| Repeated Switches (Fowler) | the same switch/if-cascade on the same type recurs across the change → replace with polymorphism, or one map both sites share |
| Shotgun Surgery (Fowler) | one logical change forces scattered edits across many files in the diff → gather what changes together into one module |
| Divergent Change (Fowler) | one file or module is edited for several unrelated reasons → split so each module changes for one reason |
| Speculative Generality (Fowler) | abstraction, parameters, or hooks added for needs the spec doesn't have → delete it; inline back until a real need shows |
| Message Chains (Fowler) | long `a.b().c().d()` navigation the caller shouldn't depend on → hide the walk behind one method on the first object |
| Middle Man (Fowler) | a class or function that mostly just delegates onward → cut it, call the real target direct |
| Refused Bequest (Fowler) | a subclass or implementer that ignores or overrides most of what it inherits → drop the inheritance, use composition |
| **Honest mechanism (anti-hack)** | *kept verbatim below* |

**Overlap with the architecture lenses is expected — report a finding once, under whichever name fits better, never both.** Nearest pairs: Naming honesty / Mysterious Name; Logic consolidation / Duplicated Code & Repeated Switches; Reinvention & dead weight / Speculative Generality; Information hiding vs relabeling / Middle Man.

**Honest-mechanism lens (verbatim).** Fix the cause, not the symptom. The smell is a fix that **infers structure from surface form** — a regex guessing paragraph breaks, a magic-string special-case, a heuristic that passes the one observed input. These pass the demo and rot on the next input, because the defect is still upstream where the data first went wrong. Ask, per fix in the diff: is it applied at the boundary where the bug was *noticed*, or at the layer where the value first becomes wrong? Does any change reconstruct lost information by guessing (regex on formatting, parsing a rendered string, sniffing a magic substring) instead of preserving it at the source? Does a special-case branch exist only to make one observed input pass — what's the next input that breaks it? **Revision:** move the fix upstream to where the value is first wrong; delete the surface-form heuristic and preserve the real signal at the producing layer. A genuinely-warranted symptom-patch is tagged `[symptom-patch]` with the tradeoff surfaced — never shipped as a root-cause fix (`~/.claude/CLAUDE.md` "Fix discipline").

Returns a consulting report: per finding a named **Challenge** (cite files/line ranges, never paste contents), a **Specific revision** the reader can execute without further design work, and a **severity** — **Critical** (concrete pain within 1–2 phases) / **Worth-considering** (long-term friction) / **Nit** (stylistic). Every applicable lens leaves a trace; clean lenses list under "What I attacked but found OK." An empty report = failed run. **Triage:** Critical → HIGH (fix loop); Worth-considering → fix-if-cheap single-file, else `T-N [polish]`; Nit → backlog/ignore. Reinvention-lens findings map the same way, with two carve-outs: a delete/stdlib/native/inline that crosses an API-contract or public-interface boundary needs a design decision → log the skip with a one-line reason, don't auto-apply; a purely line-level shrink → fix if single-file/no-contract, else `T-N [polish]`.

**1c — Correctness pass (Opus subagent).** Target = `main..HEAD`. The architecture pass judges *shape*; this hunts *behavior* — the bug a careful reader catches that tsc, the unit tests, and one-curl-per-contract miss. Default stance: assume a reachable input breaks it until you've traced the paths. **Every finding names a concrete failure — a specific input or state → the wrong output or crash — verified against the actual code; a finding you can't reduce to a failing scenario is a nit, not a bug.** No speculation, no "consider handling"; don't re-flag what tsc or an existing test already guards.

Bug classes (skip one only when the diff can't reach it): input validation at trust boundaries; error/exception paths (swallowed errors, no rollback, partial writes); null/undefined/empty/zero; boundary & off-by-one; auth & access control (missing ownership check, IDOR); injection/escaping (SQL, shell, HTML); concurrency, races & double-submit; resource cleanup (unclosed handles, leaks); async correctness (unawaited promise, unhandled rejection); wrong conditional/logic (inverted guard, wrong operator).

Returns per finding: a **Bug** (cite file/line range, never paste contents), the **failure scenario** (input/state → wrong result), a **root-cause fix direction** (where the value first goes wrong — never a surface-form patch), and a **severity**. **Gates:** **HIGH** — data loss/corruption, auth/authz bypass, a crash or wrong result on the *primary* user flow, any money/security path → fix loop. **MEDIUM** — wrong behavior on an edge/error path off the primary flow, missing error handling that loses user work, a leak under normal use → fix loop. **LOW** — defensive-only on an unreachable path, or a nit → backlog/ignore. An empty/all-LOW return is valid only if the pass names the flows it actually traced; a reflexive "looks correct" with nothing traced = failed run.

**1d — Re-verify.** Any architecture- or correctness-pass fix applied inline changes code with no behavioral gate of its own — re-run 1a's checks. Any new `✗` = HIGH.

**Gate:** Round 2 opens only when 1a + 1d are green. 1b/1c surviving must-fix findings merge into the **same** fix loop as dogfood findings — never a second loop.

### Round 2 — Dogfood (skip entirely if `ui: false`)

One continuous `/browse` session, empty → populated. Write `currentSubStep: "review.dogfood"`. **Browser-optional ladder — drive `/browse` only where it earns it:** an outcome already proven at Round 1's automated rung (tsc, unit/component tests, one curl per API contract) is not re-driven in the browser; the guided walk browser-verifies only the last mile a test can't — that the UI is *wired* to the behavior and *visibly confirms* success. A pure data/logic outcome with no new UI to exercise → assert it at the cheapest sufficient rung, no browser.

1. **Blind first-impression** (Engine 2, restricted context) — empty app. Feature phases add: name the new screens, explore them, confirm they're reachable from existing nav and you can return to the main app.
2. **Guided walk** (Engine 1, single **stateful** subagent — seeding data through the UI *is* the primary flow; for stories needing prior state, seed via API/DB). Walks, in one session:
   - Every user story in **this** phase's `requirements.md` (never re-walk prior phases).
   - Every manual check in `validation.md`.
   - **Phase 1+ shell regression (hardcoded, always runs regardless of `validation.md`):** (1) global nav renders + interactive on a Phase-0 screen; (2) logo/home reaches the dashboard from a route added this phase; (3) auth still gates Phase-2 routes — unauthenticated → login redirect; (4) toast fires on a Phase-2 action; (5) any nav item added this phase is in the right place and clickable.
   - **Screen presence:** any screen named in `requirements.md` no story exercised — confirm it renders (no 404, blank, or JS error overlay).
   - Per item: navigate → act → screenshot → assess *does the outcome match the promise AND did the system confirm success visibly* (not "returned 200"). Record ✓ / ⚠ (works but no feedback / confusing / dev language) / ✗ (seen vs expected). `✗` = HIGH, `⚠` = MEDIUM. **No item passes without a screenshot proving the outcome state.** Delete seeded test data after if it pollutes the user's real board.
3. **Findings triage + outcome-card grading** (main, text). Consolidate both sources from text reports + screenshot paths; dedupe; assign final severity. Any finding landing on a surface outside this phase's card is `expected-not-built` (LOW, informational) — never HIGH/MEDIUM, never a fix-loop trigger; main makes this call from full context, never the blind reviewer. Open a screenshot only when a verdict is genuinely ambiguous.

   **Outcome-card grading (verbatim intent).** Using the blind reviewer's observations and the guided walk's recorded outcomes, grade each PRIMARY outcome: the card outcome verbatim, delivered recognizably Yes/Partial/No, and the on-screen signal seen vs the card's "Success looks like" line. `Partial`/`No` on any primary outcome → **HIGH**. If `design-brief.md ## Design intent` has a memorable-thing line, judge whether the experience delivered that feeling — `Partial`/`No` → **MEDIUM**. Legacy phase with no card → grade the user goal from `requirements.md`. **Grade caller-facing promises by what a caller can actually obtain, not by what the code names or caps internally.**

---

## standalone-dogfood

The verification gate for any implementation outside `/build`. Uses the app via `/browse`, never by reading code. **Reads no `.build-state.json`, requires no spec dir, writes nothing to build state** — organic/ad-hoc invocation ("done", "just built X"). **Exception:** when the orchestrator explicitly invokes this mode to close out a narrow-ceremony phase (`phaseCeremony: "narrow"`), steps 1-6 below run exactly the same, but on a passing gate write `.build-state.json` `step: "phase-complete"` (or `"phase-blocked"` on Stop) — the same terminal write pipeline-review owns, just from this mode. Never report-only.

1. **Find the running app.** `lsof -i :3000 -i :3001 -i :5173 -i :8080 -i :4000 -i :8000 2>/dev/null | grep LISTEN` → use that port. If none, read `package.json` `scripts.dev`/`scripts.start` or `CLAUDE.md`, start it in the background, wait ≤15s. Still nothing → ask once for the URL.
2. **Two distinct sentences.** From (in order) the arg, `specs/*/requirements.md` (most recent dated dir), `git diff HEAD~1 --stat` + `git log -1 --format=%B`, else ask once: write an **Original problem** (what the user couldn't do / kept hitting / worked around) and **What was built**. If the two read the same, you're verifying the implementation, not the problem — re-derive.
3. **Derive 2–4 scenarios from the problem, not the implementation.** Bug fix: scenario 1 reproduces the situation that triggered the bug and verifies it's gone. New feature: scenario 1 = "user arrives with the problem, tries to solve it naturally"; scenario 2 = one edge case (empty state, validation failure, error) if applicable. Classify each **Simple** (navigate → verify present/absent/correct) or **Flow** (action → verify intermediate → action → verify outcome); document each before executing.
4. **Run the shared engine.** Engine 1 with the classified scenarios, then Engine 2 blind reviewer with the *Original problem* sentence as the goal (blind rule above — nothing else). Blind findings are first-class; **avoid double-counting** — the blind pass runs *after* scenarios pass, so its findings should be NEW gaps.
5. **Three-signal gate** (Engine 3). Do not output "done"/"complete" until all three clear, even if the user said "finish up." Signal #2 fail → HIGH → fix loop. **Signal #3 only (1+2 passed)** → surface a user decision: functionally complete, blind reviewer flagged [list], fix now or accept and move on?
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
- **Round 1 — Code review** — checks N/N; architecture N Critical / none (M simplifications, −K lines); correctness N HIGH / none.
- **Dogfood — Blind first impression** — what the blind reviewer found, or that the goal was reached cleanly.
- **Dogfood — Story & shell coverage** — story/check table; screen presence N/N; shell regression (Phase 1+) PASS or failures.
- **Auto-fixes Applied** — one line per fix, or "None."
- **Phase Verdict** — 1–2 sentences: did the phase deliver its card contract, ready for handoff?

(standalone uses the shorter closing report from its step 6 instead.)

---

## Brain integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md` with `$AGENT=review`, `$TAGS` from `tech-stack.md`. **Friction:** one entry per real bug — `Phase <N> review friction: <bug name>`; body = which sub-pass surfaced it, which story, how resolved, what check would catch it earlier. **Phase-wrap:** once after the final report — `Phase <N> review: <summary>`; body = what caught real bugs, what UX pattern to watch next.

---

## Ground rules

- **One engine, two scopes.** Both modes call `browser-review-engine.md` — no second dogfood implementation exists. They differ only in scope (pipeline adds shell regression + outcome grading) and terminal write (pipeline writes `phase-complete`/`phase-blocked`; standalone writes nothing — except a narrow-phase closure, which writes the same terminal state).
- **`/browse` only** — never Puppeteer, curl, or reading code to infer appearance. Screenshot every story outcome + meaningful screen at 1280px AND 390px. "Checked the code" is not UX verification.
- **Round 1 checks green before the browser (pipeline).** Architecture + correctness findings merge into the SAME fix loop as dogfood findings — never a second loop.
- **All HIGH + MEDIUM auto-fix silently (pipeline); a felt-impact fork is never a silent pick** — surface it. Agents never commit; this skill commits. Cap 3 → Accept/Stop binary, no further attempts unless the user asks.
- **Terminal state before any user-facing output (pipeline); standalone writes no build state.** User is non-technical — they see only the final report, the cap-hit binary, or a fork. Never ask which bug to prioritize or whether a deviation matters.
