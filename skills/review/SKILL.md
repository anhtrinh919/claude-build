---
name: review
description: >
  SDD review skill — two sequential steps gated by pass/fail. Step 1: validation.md compliance (runs every automated check, then every manual verification from the spec's validation.md — binary pass/fail). Step 2: UX dogfooding (only if Step 1 passes — navigates the app as a real user using the browse skill, looking for broken flows, confusing UX, missing feedback, empty state quality). Build pipeline only — invoked by /build when the backend-complete step is reached. Never invoked standalone or outside the /build pipeline. For testing outside the build pipeline, use /dogfood instead.
---

# /review — Spec Compliance & UX Dogfooding

Input: `specs/YYYY-MM-DD-[feature]/validation.md` — this is the test contract. Read it first.

If no `validation.md` is found: stop. "No validation spec found. Run `/spec` for this phase first."

Step 1 must pass before Step 2 starts. No exceptions.

## Invocation contract

| Mode | Model | Mechanism | Inputs (caller passes) | Outputs (skill produces) | Terminal state (this skill writes) |
|---|---|---|---|---|---|
| (single mode) | Sonnet subagents for test execution + report writing; main session (Opus default) for orchestration + code fixes | **inline** (in `/build` session) | spec directory, validation.md, outcome-card.md, mission.md (for design-tool track), running app | clean review report (success) OR cap-hit binary surface to user (failure) | `phase-complete` (clean convergence OR user picks Accept on cap-hit); `phase-blocked` (user picks Stop on cap-hit) |

**Why inline:** this skill orchestrates its own subagents (reviewer fleet, walk chunks, report writer) and its fix-loop agents must inherit the session's model — subagents can't spawn subagents (one-level nesting rule, `${CLAUDE_PLUGIN_ROOT}/skills/_shared/subagent-policy.md`), so running this skill as a subagent silently breaks both.

The auto-fix loop (see "Auto-fix loop policy" below) is owned entirely by this skill — never surfaces technical decisions to the user except at the cap-hit binary. `reviewIteration` counter persists in `.build-state.json` across loop iterations.

---

## Model split

Pinning the cheaper model on the right work keeps cost predictable when this skill is invoked from an Opus session inside `/build`.

| Stage | Runs on | Why |
|---|---|---|
| Step 1 manual verification (browse passes) | **Sonnet subagent** | Execution work — navigate, observe, record. No architectural reasoning. |
| Step 1.5 visual compliance (design ↔ built diff) | **Sonnet subagent** | Side-by-side comparison is execution work. |
| Step 2 reviewer fleet (Pre-2a.5) | **Three Sonnet subagents** (Engine 2, persona variants) | Blind first-impressions — fresh perspectives, not deep reasoning. |
| Step 2 story walks + screen review (2a–2f) | **Sonnet subagent** | Same as Step 1 manual — execution, not reasoning. |
| Step 2 report writing (the big structured report) | **Sonnet subagent** | Synthesis into a fixed template is execution work. |
| Auto-fix loop dispatch (code edits + `/code-harness`) | **Subagent inheriting the main session model** (Opus by default) | Root-cause analysis + spec-correct fixes need full reasoning. |
| Final user-facing surface (cap-hit binary, terminal state writes) | **Main agent inline** | These are orchestration decisions, not work. |

Anywhere this skill says "spawn a subagent" without naming a model, default to **Sonnet** unless the stage is in the auto-fix loop row above. Anywhere it says "do X" (no subagent), the main agent runs it.

---

## Auto-fix loop policy

This skill runs in a loop until all checks pass or the iteration cap is hit. Anywhere this skill says **trigger auto-fix loop**, apply the policy below — never stop and ask the user to make a technical decision about HIGH/MEDIUM bugs, broken validations, or visual drift. The user is non-technical: their role is binary (accept-as-is, or stop), and they only see the loop when the cap is hit.

**Iteration counter:** Read `.build-state.json` in the project root. Use the `reviewIteration` field (default 0 if missing — see the typed schema in `${CLAUDE_PLUGIN_ROOT}/skills/build/SKILL.md`). Increment by 1 every time the loop fires; write the file back before dispatching, preserving every other field (`phase`, `feature`, `step`, `requirementsHash`, `currentSubStep`, `dogfoodPid`).

**Loop body — apply whenever a step fails or Step 2 surfaces HIGH/MEDIUM issues:**

1. Collect the current failures into a brief: the failing automated checks (with output), the drifting frames (with frame IDs and what differs), or the HIGH/MEDIUM Step 2 issues (with the table rows verbatim). Include the verify command that must pass for each.
2. Spawn a subagent via the `Agent` tool with `subagent_type: general-purpose`. Brief it with:
   - Phase spec paths: `requirements.md`, `plan.md`, `handover.md`
   - The exact failing checks/issues to fix — paste from the current run, do not summarise
   - The verify commands that must pass after the fix (e.g. `bash specs/.../verify-group-N.sh`, the failing manual check, the failing user-story walk)
   - **Hard constraint — invoke `/code-harness` for every fix.** Each fix must go through code-harness's discipline pipeline: write/update a verify-script first (or use the existing `specs/.../verify-group-N.sh`), confirm the verify-script is red before implementing, then implement and confirm it passes. The subagent is not allowed to commit a fix without a verify-script run that passes.
   - Hard constraints: fix only what is flagged; do not refactor, do not extend scope, do not change the spec; re-run the failing verify after each fix and stop only when it is green; root-cause iron law applies (no surface-patching tests to make them pass); if a fix needs more than **3 hypotheses**, return with the diagnosis instead of pushing further (matches the unified retry cap).
   - **List the files the subagent touched** in its return summary — used for the regression-band scope below.
3. Wait for the subagent to return. Read its summary — note which fixes landed, which didn't, and which files were touched.
4. **Scoped re-validation** (not a full re-entry from Step 0). Run only:
   - Every check that was failing in step 1 above. Validate from scratch — never trust the subagent's "fixed" claim.
   - **Regression band:** any user story whose handler shares a file with the touched-files list. (Cross-ref `requirements.md` user stories against the file list. If unsure, default to walking the story.)
   - **Skip the reviewer fleet (Pre-2a.5) unless the fleet signal itself was the failure being fixed** — and then re-verify with ONE fresh reviewer (the persona that failed), not all three. Spawning fresh reviewers on every loop iteration is wasted cost — when only Step 1 / 1.5 / 2a–2f checks were failing, the fleet's previous verdicts still stand until the last fix lands.
   - Skip Step 0 (scope drift detection) — the diff hasn't fundamentally changed; only fix-touched files have moved.
5. **Last-fix-lands rule:** when scoped re-validation passes for the last failing check (no failures remain across the loop), run a **final full pass** — Step 1 fully, Step 1.5 fully, Step 2 (2a–2f) fully, AND the reviewer fleet fresh (all three personas). This is the "nothing else regressed" check. If the final full pass surfaces new failures, increment `reviewIteration` and re-enter the loop body. If it passes clean, write the convergence terminal state per "Reset + clean-convergence" below.

**Cap:** If `reviewIteration >= 3` and there are still failures after the fresh run, stop the loop and surface once to the user as a binary:

```
Phase [N] review: [X] issue(s) still failing after 3 fix attempts.

Still failing:
- [issue 1 — one line]
- [issue 2 — one line]

Accept anyway (mark phase complete with known issues), or Stop (call this phase blocked)?
```

Wait for the user's reply. No further fix attempts unless the user explicitly asks for another round.

**Cap-hit terminal state writes:** when the user replies to the binary, write `.build-state.json` with the matching terminal `step` before returning to `/build`:
- User picks **Accept anyway** → write `step: "phase-complete"`, reset `reviewIteration` to 0, null `currentSubStep`, preserve all other fields.
- User picks **Stop** → write `step: "phase-blocked"`, reset `reviewIteration` to 0, null `currentSubStep`, preserve all other fields.

This is the resume anchor for the cap-hit branch — without it, a compaction after the user's reply leaves the next `/build` confused about whether the phase blocked or accepted, and forces another review pass.

**LOW-only outcome:** If Step 2 produces only LOW-severity items, do not loop. Fix trivial ones silently in the same run; log the rest in the report; complete normally.

**Reset + clean-convergence terminal write:** When the loop completes cleanly (Step 2 ends with no HIGH/MEDIUM items), write `.build-state.json` with `step: "phase-complete"`, reset `reviewIteration` to 0, null `currentSubStep`, preserve all other fields — do this **before** writing the final report. This is the resume anchor for the convergence branch. If context is compacted between report-render and `/build`'s post-review handling (dogfood handoff, phase wrap), the next `/build` reads `phase-complete` and resumes the next-feature path instead of re-running the whole review.

If the user looks at the rendered report and asks for further fixes, that's a fresh `/review` invocation — `step` will be moved off `phase-complete` by the new run's increment of `reviewIteration`.

**Sub-skill breadcrumb:** On entry, write `currentSubStep: "review.<step>"` to `.build-state.json` (e.g. `review.step-1`, `review.step-1.5`, `review.step-2`). Null it out on clean exit. `/build` reads this on resume.

**What the user sees:** nothing about iterations, subagents, or fix attempts — until the cap is hit. Each loop is internal. The final output is either a clean report (success) or the binary surface above (cap hit).

---

## Wiki integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/_shared/wiki.md` with `$AGENT=review` and `$TAGS` from `tech-stack.md`.

**Friction trigger:** for each bug found during Step 1 (failed automated or manual check) or Step 2 (UX/visual issue). One entry per bug. Title: `Phase <N> review friction: <bug name>`. Body: where the bug surfaced, which user story it touched, how it was resolved, what check would have caught it earlier.

**Phase-wrap trigger:** once at the end of Step 2, after the report is written, before user approval. Title: `Phase <N> review: <one-line summary>`. Body: which automated checks caught real bugs, which manual checks surfaced issues, what UX pattern to watch next time.

---

## Step 0 — Scope Drift Detection

Before running any checks, compare what was specced against what was built.

1. Read `plan.md` task groups.
2. Run `git diff main..HEAD --name-only` (or equivalent for the feature branch) to see what files changed.
3. For each task group, classify as:
   - **DONE** — files for this group are present in the diff with expected changes
   - **PARTIAL** — some files changed but key pieces appear missing
   - **NOT DONE** — no relevant files in the diff for this group
   - **SCOPE CREEP** — files changed that are outside all task groups

4. Surface a one-line table before proceeding:
   ```
   Group 1: DONE
   Group 2: PARTIAL — missing [what]
   Group 3: NOT DONE
   ```

**Gate:** If any task group that covers a primary user flow story is NOT DONE or PARTIAL, **trigger auto-fix loop** with the incomplete task group(s) as the brief. Do not run Step 1 in this iteration.

Non-primary groups that are NOT DONE: surface as a warning but do not trigger the loop.

---

## Step 1 — Validation.md Compliance

**Step 0 (wiki):** Run Read wiki (see Wiki integration) before running automated checks.

### Automated checks

Run every automated check listed in `validation.md`. For each:

1. Execute the command exactly as specified
2. Record exit code and output
3. Report: `✓ [check name]` or `✗ [check name] — [what failed]`

The dev server must be running before API checks. Start it if needed; poll until ready (max 30s). Never ask the user to start a server.

After all automated checks:
- All pass → proceed to manual verification
- Any fail → **trigger auto-fix loop** with the failing checks (and their output) as the brief. Do not proceed to manual verification or UX review in this iteration.

### Manual verification — Sonnet subagent runs the walk

Dispatch the manual-verification walk to a **Sonnet subagent** (`Agent` tool, `subagent_type: general-purpose`, `model: "sonnet"`). The brief contains every manual check from `validation.md` verbatim plus the running app URL + credentials.

**Subagent brief shape:**

```
You are running validation.md manual checks via /browse. Tools: /browse only. No source access, no file edits.

Running app URL: <URL>
Login (if needed): <credentials>

Checks (run each one clean pass, screenshot every meaningful state change):
1. <check from validation.md verbatim>
2. <check from validation.md verbatim>
...

Per check: navigate → action → observe → record one of:
- ✓ [check name]
- ✗ [check name] — [what was seen vs what was expected]

Output ONLY a list in that format. No commentary, no suggestions, no fix attempts.
```

Main agent reads the returned list.

After all manual checks:
- All pass → Step 1 complete, proceed to Step 1.5 (or Step 2 for non-UI phases)
- Any fail → **trigger auto-fix loop** with the failing checks (and what was seen vs expected) as the brief. Do not proceed to UX review in this iteration.

### Step 1 completion report

```
Step 1 — Validation Compliance
Automated: [N]/[N] passed
Manual: [N]/[N] passed
Status: PASS / FAIL
```

---

## Step 1.5 — Visual compliance (UI phases only) — track-aware

Only runs after Step 1 fully passes. **Skip this step explicitly when `requirements.md` frontmatter has `ui: false`.** If the `ui` field is missing, default to `true` for backward compatibility (existing phases without the flag are assumed UI). Never infer from screen count — the flag is the only signal.

Purpose: confirm every screen described in the handover was actually built as designed. This catches the failure mode where code runs, tests pass, but the built screen looks nothing like the design.

**Read `mission.md` `## Design Tool` first** to determine the track. Apply legacy-value mapping per `/frontend` backward-compat (`external` → `external-pencil`; `claude-code` / `claude-code-taste` → `claude-code-impeccable`). The diff target depends on the track.

**Dispatch the per-track comparison work below to a Sonnet subagent** (`Agent` tool, `subagent_type: general-purpose`, `model: "sonnet"`). The main agent picks the track and assembles the brief; the subagent does the navigation, screenshots, comparisons, and returns the structured row list. The subagent inherits whichever toolset the track requires (Pencil MCP for external-pencil; `/browse` for both built-app capture and Claude Code mockup capture).

### External tracks (`external-pencil` / `external-other`)

1. **Open the design file** using the path in `handover.md` "Design file" section. Pencil: `mcp__pencil__open_document` → `mcp__pencil__get_editor_state`.
2. **Iterate the frame index.** For every row in the handover frame index:
   - Navigate the running app to the state that row describes. Use seeded fixtures or direct URL navigation as needed.
   - Capture the built screen via `browse` / Puppeteer / MCP browser. Match the viewport listed in the frame (1280px desktop, 390px mobile).
   - Capture the design frame via `mcp__pencil__get_screenshot({ nodeId })` (or the tool equivalent).
   - Compare side-by-side. Record one of:
     - ✓ Match (structural match — minor pixel differences are OK)
     - ⚠ Drift (wrong chrome, wrong palette, missing element, extra element — specify what)
     - ✗ Not built (the state doesn't render at all)

For non-Pencil design tools, use the equivalent MCP/export. If no machine-readable design file exists (screenshots only), degrade to human review: post built screenshot + designer-provided screenshot side-by-side and ask the user to confirm match.

### Claude Code track (`claude-code-impeccable`)

The "design" lives in `specs/<this phase>/mockups/` as static HTML mockup files. There is no Pencil MCP call here.

1. **Iterate the mockup index** in `handover.md` (the section listing each mockup file and the screens it covers).
2. For every screen + state covered by a mockup file:
   - Render the mockup. Open the mockup file in the same browser (or a static server if the project requires it). Capture via `browse` at the spec viewport (1280px desktop, 390px mobile).
   - Render the built app at the same state. Navigate the running app, seed state if needed, capture via `browse` at the same viewport.
   - Compare side-by-side. Record one of:
     - ✓ Match (structural match — minor pixel differences are OK)
     - ⚠ Drift (wrong chrome, wrong palette, missing element, extra element — specify what)
     - ✗ Not built (the state doesn't render at all)

If a state is in `requirements.md` but not in the mockup index, that's a `/frontend` handover gap — fail Step 1.5 and surface to the auto-fix loop with the gap (the loop will return work to `/frontend`).

### Summary (both tracks)

```
Step 1.5 — Visual Compliance
Track: <track name>
✓ Match: [N] screens
⚠ Drift: [N] screens — [list]
✗ Not built: [N] screens — [list]
Status: PASS / FAIL
```

Gate: any `⚠ Drift` or `✗ Not built` rows fail this step. **Trigger auto-fix loop** with the drifting/missing screens as the brief. Do not proceed to Step 2 in this iteration.

---

## Step 2 — UX Dogfooding

Only runs after Step 1 fully passes.

**Execution dispatch.** Step 2 has four execution chunks that all run as **Sonnet subagents** (`Agent` tool, `subagent_type: general-purpose`, `model: "sonnet"`):

1. **Pre-2a.5 reviewer fleet** — three blind Engine-2 reviewers, persona variants (see Pre-2a.5).
2. **2a story walks** — one Sonnet subagent walks every user story end-to-end and returns the coverage table rows.
3. **2b–2e screens / mobile / edge / nav** — one Sonnet subagent does all the screenshot-and-assess work and returns visual findings.
4. **2f problem-resolution + design-intent check** — one Sonnet subagent does the cold-start walk and returns the verdict block.

The main agent orchestrates: persona reset → dispatch fleet (blocking) → dispatch 2a → dispatch 2b–2e → dispatch 2f → collect all findings → dispatch the report-writer Sonnet subagent (see "Step 2 report" below). Each subagent gets the relevant context (URL, credentials, the specific stories/screens for its chunk) plus the persona-reset sentences — never the implementation, never the diff, never `plan.md`.

**Chunks 2a → 2b–2e → 2f run sequentially by design — do NOT parallelize them.** They share one app instance with mutable seeded state: 2a mutates state while walking stories, 2b–2e screenshots would capture mid-mutation states, and 2f requires a blank cold-start. The fleet's internal concurrency rule (Pre-2a.5) is the only sanctioned parallelism in Step 2.

You are a meticulous, opinionated power-user doing QA. You care about: things actually working, clear feedback, sensible defaults, no dead ends, no broken states. You are NOT reviewing code quality — you are reviewing the experience for humans.

### Persona reset — anchor on the outcome card AND the design intent

Before any browser work, do two reads:

1. **Read `specs/<this phase>/outcome-card.md`** — the user-approved contract. Extract the **phase goal** and the **primary outcomes with their "Success looks like" signals**. Write the goal as one sentence:
   > "I am a [user type] trying to [phase goal]. I am opening this app for the first time."
   
   Keep the outcomes + signals list at hand — 2f grades each one explicitly. (Legacy phase with no card: fall back to extracting the user goal from `requirements.md` as before.)

2. **Read `specs/<this phase>/design-brief.md` `## Design intent`** and extract the **memorable thing** the experience is meant to leave the user with (the answer the user gave when /frontend asked "what's the one thing a user should remember or feel after using this?"). Write it as one sentence:
   > "The experience should leave them feeling [memorable thing]."
   
   If the design brief doesn't exist or has no `## Design intent` section (e.g. backend-only phase, legacy phase), skip the memorable-thing line — Step 2 then grades on the user goal alone.

Hold both frames throughout Step 2. You are not checking the implementation against the spec — you are experiencing the product as someone who has the original problem, has never seen the solution, and is supposed to walk away feeling the memorable thing.

### Browser setup

Locate the browse binary. Check for the gstack browse binary first; fall back to the system browse if available:
```bash
B=~/.claude/skills/gstack/browse/dist/browse
if [ ! -x "$B" ]; then
  B=$(which browse 2>/dev/null || echo "")
fi
if [ -n "$B" ] && [ -x "$B" ]; then echo "BROWSE_READY: $B"; else echo "BROWSE_MISSING"; fi
```

If missing, rebuild: `cd ~/.claude/skills/gstack && bun install && cd browse && mkdir -p dist && bun build src/cli.ts --compile --outfile dist/browse`

Do not continue without a working browse binary. Do not fall back to Puppeteer or curl — ever.

If the app redirects to login, check `.env`, `.env.local`, or `CLAUDE.md` in the project root for credentials, then log in before proceeding.

**Take a screenshot after every meaningful action.** Do not skip screenshots to save time.

---

### Pre-2a: State Setup

Before walking user stories, identify which stories require live run state (any story involving: a run in progress, a gate pending, a failure, a past run log). For those stories, seed the required state directly into the database using the API or direct DB insert. This is mandatory — do not skip a story because "it needs a real run." 

**State seeding approach:**
1. Start the dev server, create a test project pointing at a temp directory if needed.
2. For `running` state stories: POST `/api/cards/:id/runs` with fake claude binary on PATH (use `app/test/fixtures/fake-claude-done.sh` for a quick run or inject manually).
3. For `gate-pending` state stories: POST run, wait for fake-claude-gate.sh to emit gate, then walk the UI.
4. For `failed` state stories: POST run with fake-claude-fail.sh, wait for `failed` status.
5. For `past run history` stories: any completed run suffices; create via API or verify existing runs.
6. For streaming animation (real-time chip arrival): browse is snapshot-only — verify the **final rendered state** (chips present, in order, correct content) rather than the animation timing. Note this caveat in the report.

Cleanup: delete test runs/projects created during review if they pollute the user's real board.

---

### Pre-2a.5 — Reviewer fleet pass (blind) — BLOCKING

Apply **Engine 2** from `${CLAUDE_PLUGIN_ROOT}/skills/_shared/browser-review-engine.md` **three times — one reviewer per persona**:

1. **First-time user, desktop** — viewport 1280px. Persona line: "You are a first-time user on a desktop computer. You have never seen this app."
2. **Impatient user, mobile** — viewport 390×844. Persona line: "You are on your phone, in a hurry, with little patience. If something isn't obvious within a few seconds, you note it and try at most once more before giving up on that path."
3. **Returning user, second visit** — viewport 1280px. Persona line: "You used this app once yesterday to accomplish this same goal. Today you're back to do it again — you remember roughly what you did but none of the details. Note anything that's harder than it should be the second time."

Each reviewer gets ONLY: its persona line, the phase-goal sentence from "Persona reset" (verbatim), the memorable-thing line from `specs/<this phase>/design-brief.md` `## Design intent` if present, the running app URL + login credentials. Identical blindness rules per Engine 2.

**Concurrency:** run the three in parallel ONLY if the app supports per-user isolation (each reviewer gets its own login/account so their actions don't contaminate each other's first impressions). Otherwise run them sequentially — three blind reviewers mutating shared state mid-pass corrupt the read. Sequential cost (~3× 2–4 min) is acceptable; contaminated findings are not.

**Aggregation (2-of-3 rule):**
- Flagged by **2+ reviewers** → confirmed finding, straight into the consolidated severity tables.
- Flagged by **1 reviewer** → run one targeted Engine-1 scenario re-check on that specific issue before it may enter the fix loop. Confirms → finding. Doesn't reproduce → log as LOW observation only. This protects fix-loop iterations from single-reviewer false positives.
- **Goal verdict for the report = the worst of the three** (any "No" beats "Partial" beats "Yes").
- **Cost rule:** the full fleet runs on iteration 1 and the final full pass only. The existing skip-naive-on-iterations-2+ rule applies to the whole fleet; if the fleet signal itself was the failure being fixed, re-verify with ONE fresh reviewer (the persona that failed), not all three.

**This step blocks 2a. Wait for all fleet reviewers to return before starting the 2a story walk.** Do NOT run 2a in parallel "to save time." 2a's whole job is to use the fleet's findings to focus what it walks more carefully and dedupe issues into a single consolidated report — if 2a starts before the fleet returns, the findings arrive too late to inform anything and every overlapping issue ends up logged twice. The wait is not dead time: while waiting, the only allowed work is *passive* preparation — listing the user stories you'll walk, confirming seeded state from Pre-2a is still intact. No active 2a story navigation, no screenshots, no story-pass/fail recording until the fleet's reports are in hand.

**/review-specific aggregation rules** (extend the engine's severity tags):

- **Findings do NOT fire the auto-fix loop independently.** They aggregate with 2a–2f results; the loop is considered once at the end of Step 2 based on the consolidated severity list. Firing on fleet findings alone would waste an iteration — story walks would catch the same issues and re-trigger.
- **Use fleet findings to focus story walks and dedupe.** When 2a runs, you already know what the fleet flagged. If a reviewer said "Story 3 area was confusing" and 2a confirms it, that is **one** issue in the consolidated report (UX Issues table), not two. The Reviewer Fleet section of the report records the high-level "did the user accomplish the goal" verdicts; specific issues live in the Bugs / UX Issues / Visual Issues tables.

---

### 2a — Walk EVERY user story (mandatory)

**This step is not optional. Every user story in `requirements.md` must be walked.** A story cannot be marked as passing without a screenshot proving it.

**Precondition — the reviewer fleet must have returned.** Pre-2a.5 is BLOCKING (see above). Do not start the 2a walk until all fleet reviewers have returned and you have read their findings. Running 2a in parallel with the fleet breaks the dedupe rule and the focus-the-walk rule. If you find yourself about to navigate to a story while a reviewer is still in flight, stop — wait, then walk.

For each user story in `requirements.md`, in order:

1. **Read the story** — understand exactly what action it describes and what outcome it promises.
2. **Set up the starting state** — if the story requires a run in a specific state, seed it first (see Pre-2a above).
3. **Navigate to the starting point** via browse.
4. **Perform the action** the story describes.
5. **Screenshot the result.**
6. **Assess functional correctness:** does the outcome match what the story promises?
7. **Assess experience quality** (mandatory — not optional). A story that works technically but fails any of these is a UX issue, not a pass:
   - Was it obvious what to click / type / do at each step? Or did you have to think?
   - Did the system confirm success in a way the user can recognize? (Not just "the request returned 200" — visible confirmation: a banner, a state change, a checkmark, the new item appearing.)
   - Would a first-time user — with the original problem and no inside knowledge — feel confident the goal was achieved?
   - Did anything use developer language (raw IDs, status codes, "submitted") instead of human language?
   - On the path you walked, were there any moments where a real user would pause, re-read, or guess?
8. **Record:** `✓ Story N: [one-line description]` (functional pass + experience pass) or `⚠ Story N: works but [experience issue]` (functional pass, experience fail) or `✗ Story N: [description] — [what was seen vs expected]` (functional fail). `⚠` rows go into the UX Issues table at MEDIUM severity minimum.

**If a story cannot be demonstrated** (missing feature, broken state, or impossible to reach): mark it `✗` and document the blocker. Do NOT skip it silently — a skipped story is a failed story.

**Stories that involve real-time behavior** (streaming, polling, toast timing): seed the final state, screenshot the result, note "animation not verified — state verified."

After all stories:
- All pass → proceed to 2b
- Any fail → document in report. If a failure is a blocker for the primary user flow, note it as a STOP. If it's a secondary flow, continue and surface in report.

---

### 2b — Core screens

**Load the project's design lens first.** Read `mission.md` `## Design Tool`. Map the value to a design skill and load it via the `Skill` tool before reviewing screens — review through the same lens the design was built with:

- `claude-code-impeccable` → `Skill: impeccable`
- `claude-code-taste` (legacy) → `Skill: impeccable` (per /frontend backward-compat — the engineering-taste track was removed)
- `external-pencil`, `external-other`, or missing → no skill load; fall through to the generic checks below

The loaded design skill stays active for 2b–2e. Visual issues it would flag in design (typography crimes, weak hierarchy, generic AI patterns, lazy spacing, color drift) are now reviewed against the *built* screens — not the mockups.

For each screen in `handover.md`:
```bash
$B goto <APP_URL>/<route>
$B screenshot /tmp/review-[screen]-desktop.png
```
Read each screenshot and assess visually:
- First impression — does it look finished, or does it look like a wireframe with content shoved in?
- Is the visual hierarchy obvious — can you point to "the one thing" the user should look at first?
- Is typography readable (size, contrast, line length)? Does it match the lens's standards?
- Are there misaligned elements, clipping, orphaned text, or "AI-default" patterns the lens would reject?
- Anti-pattern scan: 3+ similar-weight CTAs competing? Cards-inside-cards-inside-cards? Hero buried by chrome? Empty space that reads as "broken" rather than "intentional"?

### 2c — Mobile viewport

```bash
$B viewport 390x844
$B goto <APP_URL>
$B screenshot /tmp/review-mobile.png
```

Assess: is the layout intentionally adapted, or just squished? Text overflow, horizontal scroll, touch targets too small, content hidden?

### 2d — Edge states

- **Empty state:** Navigate to a state with no data. Screenshot. Is it clearly designed, or just a blank void?
- **Error state:** Trigger a validation error. Screenshot. Is the error message visible and readable?
- **Loading state:** If async operations exist, observe the loading state. Does it appear and resolve?

### 2e — Navigation & dead ends

Click through the main navigation. Screenshot each destination. Do active states update? Are there dead ends with no way back?

---

### 2f — Outcome card grading + design-intent check

Final check before the report. Return to the outcome card from "Persona reset" — the phase goal, the primary outcomes with their "Success looks like" signals — and (if collected) the memorable-thing sentence.

Starting from a blank state — no test fixtures, no inside knowledge of where things live — walk **each primary outcome on the card** using only what is visible on screen, as a real first-time user would. For each outcome record:

> **Outcome [N]:** "[card outcome verbatim]". **Delivered in a recognizable way?** Yes / Partial / No — [one line]. **Signal seen:** [the on-screen signal observed, vs the card's "Success looks like" line].

`Partial` or `No` on any **primary** outcome → **HIGH** finding (the phase's contract is unmet). These rows are the "Outcome Card Verdict" block at the top of the report.

Then answer four questions explicitly in the report:

1. **Discoverability:** Is the path to solving the problem discoverable from the entry screen, without guidance? If a user landed here cold, would they know where to start?
2. **Recognition of success:** When the problem is solved, does the user know it was solved? What specifically tells them — a banner, a state change, a new item appearing? "It just navigates to a new page" is not recognition.
3. **First-attempt mistakes:** What would a real first-time user most likely get wrong on their first try? Wrong button clicked, wrong field filled, missed step, wrong mental model? List the top 1–2.
4. **Memorable-thing check** (skip if no memorable-thing sentence was collected): Did the experience leave the user feeling [memorable thing]? Cite specific moments that reinforced it — or specific moments that fought against it.

Write the verdict as **two** sentences (one if no memorable-thing was collected):

> **Original problem:** "[goal sentence]". **Solved in a recognizable way?** Yes / Partial / No — [one-line reason].
>
> **Design intent:** "[memorable-thing sentence]". **Did the experience deliver that?** Yes / Partial / No — [one-line reason citing specific moments].

Severity:
- `Partial` or `No` on problem resolution → **MEDIUM** finding minimum, feeds the auto-fix loop.
- `Yes` on problem with caveats from question 3 → **MEDIUM** UX issue.
- `Partial` or `No` on the memorable-thing check → **MEDIUM** UX issue. (Not HIGH — design intent is a polish gate, not a blocker.)

---

## Step 2 report — Sonnet subagent writes it

After all four Step 2 execution chunks have returned, **delegate the report write-up to a Sonnet subagent** (`Agent` tool, `subagent_type: general-purpose`, `model: "sonnet"`). The main agent collects raw findings; the subagent merges, dedupes, severity-tags, computes the health score, and produces the structured report.

**Subagent brief shape:**

```
You are synthesising SDD phase review findings into the Step 2 report. No tools — text only.

Persona reset:
- Phase goal: <goal sentence verbatim>
- Outcome card primary outcomes + signals: <verbatim>
- Memorable thing (omit if none): <memorable-thing sentence verbatim>

Raw findings:
- Reviewer fleet (3× Engine 2 output, persona-labelled): <verbatim>
- 2a story walks (per-story PASS/⚠/✗ rows): <verbatim>
- 2b–2e screen / mobile / edge / nav findings: <verbatim>
- 2f outcome-card grading + design-intent verdict block: <verbatim>

Apply the severity rules and health-score deductions from this SKILL.md ("Issue severity + health score" section, pasted below):
<paste the rules block verbatim>

Produce the report in the EXACT structure from "Step 2 report" in this SKILL.md (template pasted below). Dedupe issues where the same root cause hit multiple chunks. Compute the health score. Write the overall verdict.
<paste the report template>

No prose outside the template. No fix suggestions — that's the main agent's job via the auto-fix loop.
```

Main agent reads the returned report and uses it to (a) trigger the auto-fix loop for any HIGH/MEDIUM issues, or (b) write `phase-complete` and return cleanly.

**Required report shape:**

```
### Review Report

**What was reviewed:** [one sentence]
**Validation:** Step 1 PASS — [N] automated + [N] manual checks
**Date:** [today]

#### Outcome Card Verdict (from 2f — this is the phase's contract)
| # | Primary outcome | Delivered? | Signal seen vs promised |
|---|---|---|---|
| 1 | [card outcome] | Yes / Partial / No | [one line] |
Any Partial/No row = HIGH finding. (Legacy phase without a card: replace this table with the Problem Resolution block below.)

#### Problem Resolution (from 2f)
**Phase goal:** "[goal sentence]"
**Solved in a recognizable way?** Yes / Partial / No — [one-line reason]
**Recognition signal:** [what specifically tells the user it worked, or "none — user has to infer"]
**Likely first-attempt mistakes:** [top 1–2, or "none observed"]

**Design intent** (omit this block if no memorable-thing sentence was collected):
**Memorable thing:** "[memorable-thing sentence]"
**Did the experience deliver that?** Yes / Partial / No — [one-line reason citing specific moments]

#### Reviewer Fleet (blind first-impressions — 3 personas)
**Goal accomplished?** first-timer: Yes/Partial/No · impatient-mobile: Yes/Partial/No · returning: Yes/Partial/No (verdict = worst)
**Top friction:** [1–3 bullets across the fleet — what confused or stopped them; tag the persona]
*(Findings already merged into Bugs / UX Issues / Visual Issues tables below; 1-of-3 flags verified or downgraded per the 2-of-3 rule.)*

#### User Story Coverage
Every story from requirements.md must appear in this table.
| # | Story (abbreviated) | Result | Screenshot | Notes |
|---|---|---|---|---|
| 1 | ... | ✓ / ✗ | /tmp/review-s1.png | ... |
...

#### Bugs
Issues that are broken or produce incorrect results.
| # | Severity | Story # | Where | What happens | What should happen |
If none: *No bugs found.*

#### UX Issues
Things that work technically but are confusing or missing feedback.
| # | Story # | Where | Issue | Impact |
If none: *No UX issues found.*

#### Visual Issues
Things that look wrong in screenshots.
| # | Where | What looks wrong | Why it matters |
If none: *No visual issues found.*

#### Improvement Ideas
Not bugs — things that would make the experience better.
1. **[Title]** — [description and why it helps]
If none: *No improvement ideas.*

**Overall verdict:** [1-2 sentences: is this ready for user's approval? What's the one thing that most needs fixing?]
```

Fix minor issues silently (copy errors, obvious visual misalignments, missing alt text). Anything classified HIGH or MEDIUM enters the auto-fix loop — never surface technical decisions to the user.

---

## Issue severity + health score

**Classify every issue** from Step 2 into one of three tiers:

- **HIGH** — broken user story, fails trunk test, missing primary flow state, user cannot complete the primary flow. Must be fixed before phase approval.
- **MEDIUM** — feature works but is confusing, missing feedback, poorly adapted on mobile, unclear error states. User decision on whether to fix.
- **LOW** — cosmetic, minor copy, alignment off by a few pixels. Fix silently if trivial, surface if it requires product input.

**Phase health score** — compute at the end of Step 2 report, before the overall verdict:

Start at 100. Apply deductions per category (deduct from that category's weighted max):
- Functional (30 pts): −8 per HIGH bug, −3 per MEDIUM bug
- Visual (20 pts): −8 per HIGH visual issue, −3 per MEDIUM
- UX feedback (20 pts): −8 per missing/broken feedback state, −3 per unclear state
- Mobile (10 pts): −8 per layout broken on mobile, −3 per minor mobile issue
- Edge states (10 pts): −8 per missing/broken edge state, −3 per partial
- Navigation (10 pts): −8 per dead end or broken nav, −3 per confusing nav

Report as: `Phase [N] health score: [XX]/100`

HIGH and MEDIUM issues both trigger the auto-fix loop — the user is non-technical and cannot meaningfully decide which technical bugs to fix. LOW issues are fixed silently if trivial, or noted in the final report. The user only sees results when (a) the loop converges to a clean run, or (b) the iteration cap hits and the binary surface fires.

---

## Ground rules

- **Model split.** Test execution (Step 1 manual, Step 1.5 visual, Step 2 chunks 1–4) and report writing (Step 2 report) run on **Sonnet subagents**. Auto-fix loop dispatches (code edits via `/code-harness`) inherit the **main session model** (Opus by default). The full table is in the "Model split" section near the top.
- Step 1 must pass before Step 2 starts. No exceptions.
- Never start Step 2 if any automated check or manual verification failed.
- **Every user story must be walked. Skipping a story is the same as failing it.**
- Always use browse — never Puppeteer, curl, or reading code to infer appearance.
- Screenshot every story. "I checked the code" does not count as UX verification.
- Use Pre-2a state seeding for any story requiring an active/past run. Never skip a story because it needs live state — seed it.
- **The reviewer fleet (Pre-2a.5) is mandatory AND blocking.** Persona-roleplay alone won't catch what a fresh user notices — you built this and can't unsee it. Each reviewer is briefed with ONLY its persona line, the phase goal, and the URL. Never pass `requirements.md` (full), `plan.md`, `handover.md`, the diff, or any implementation context. **Wait for all three to return before starting 2a story walks.** Apply the 2-of-3 rule before any fleet finding enters the fix loop. Parallel fleet execution only with per-user isolation; otherwise sequential.
- **Load the project's design skill in 2b** based on `mission.md` `## Design Tool`. Visual review uses the same taste lens as `/frontend`.
- **2f outcome-card grading is mandatory.** Every report leads with the Outcome Card Verdict table — one row per primary outcome, graded against the card's "Success looks like" signal. Partial/No on a primary outcome = HIGH. Functional pass without recognition of success = MEDIUM finding minimum.
- Rate severity honestly. Not everything is critical.
- Fix minor issues silently. **HIGH and MEDIUM go through the auto-fix loop — never escalate technical decisions to the user.**
- The user is non-technical. The only thing they ever see from this skill is a clean report (success) or the binary cap-hit surface (after 3 failed fix attempts). Never ask them which bug to prioritise, which fix to accept, or whether a deviation matters.
- Cap iterations at 3. Reset `reviewIteration` in `.build-state.json` to 0 when the loop converges.
- user starts a server or opens a browser — never. Agent owns all process and browser work.
