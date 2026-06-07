---
name: build
description: >
  Master SDD workflow — re-entrant, feature-loop aware orchestrator. State-file-first: reads .build-state.json to resume mid-phase if context was lost. Falls back to mission.md detection: missing = new project, present = next feature. New project always starts with a /bad-idea pre-flight on the user's idea brief — back-and-forth refinement until the user says "proceed with this idea" — then /ba Mode 1 → /spec Mode 1 (constitution; user approves the product story, not the files). Feature cycle: /ba (scope + user-approved outcome card) → /spec (3 spec docs validated by skeptic panel, auto-proceeds) → /frontend (design → handover; backend foundation builds in the background while the user designs) → frontend compliance check → /backend (wave-dispatched task groups with harness) → backend compliance check → /review (validation.md + reviewer fleet + outcome-card grading) → user approves. User gates are outcome-only; all technical artifacts are machine-validated. Phase boundaries stop and require a new /build invocation; within-phase gates (spec→frontend, frontend→backend, backend→review) auto-continue without stopping. Trigger on: /build, /build [idea brief], "build me X", "next feature", "start phase N", or any full-stack app request.
argument-hint: "[optional: short idea brief — only used on a new project to seed the /bad-idea pre-flight]"
---

# /build — SDD Master Orchestrator

## /eli at every gate (hard requirement)

After every `.build-state.json` write AND after every sub-skill returns, **before any further action**, invoke the `/eli` skill to produce a plain-language summary of what was just produced (the constitution, the spec, the design, the implementation, the review report, etc.). This is non-negotiable and applies to *every* gate — including within-phase auto-continue gates that don't ask the user a question.

Why it's load-bearing: the user is non-technical and tracks the project through these summaries. Skipping `/eli` after compaction is the failure mode — the agent reverts to technical talk and the user loses the thread of what's happening. The CLAUDE.md "Build Session Operating Protocol" sentinel restates this rule in always-loaded context so compaction can't drop it.

Auto-continue gates: `/eli` summary first, then continue to the next step in the same turn. Phase-boundary gates: `/eli` summary first, then ask the go/no-go question via `AskUserQuestion`.

## Voice rules

The user is not a developer. Plain language throughout — no file paths, function names, or stack traces in summaries.

- **All questions to the user MUST go through `AskUserQuestion`.** The go/no-go gate question, the user-experience fork ("I'd do X — OK to proceed?"), and any binary the orchestrator surfaces (`/review` cap-hit, compliance-check failure path, etc.) all use the structured option-picker tool. Never enumerate options inline as a numbered or bulleted list and ask the user to type back which one — that bypasses the picker UI the user expects. Plain text is for status updates, summaries, and dogfood handoff content; not for asking.
- Never ask the user to make a technical decision. When a technical fork affects user experience, present it as: "I'd do X — it means Y for users. OK to proceed?" — via `AskUserQuestion`, not inline prose.
- Phase gates are the only moments the user is asked to decide. Everything else is yours.
- Never "want me to / would you like" — make every call except the explicit gates.

---

## State detection (always run first)

**Auto-continue policy — read carefully, this is the most-misread block in the file.** There are two classes of gates and the rule is different for each:

- **Within-phase gates (`spec-complete`, `frontend-complete`, `backend-complete`) ALWAYS auto-continue, no exceptions.** These are pure resume anchors — there is no user decision here, the user has nothing to approve, and the next step is mechanically determined by the state file. Whether you are mid-session, fresh out of compaction, or a brand-new orchestrator that just woke up reading the file from disk: jump immediately to the matching resume point in the table below. Do NOT tell the user "run /build to continue" at these gates. That message is reserved for phase boundaries only.
- **Phase-boundary gates (`constitution-complete`, `phase-complete`) have a manual stop by default, BUT same-session auto-continue.** If this `/build` invocation is in the same uncompacted Claude Code session as the prior gate write (you can see your own previous gate write earlier in this conversation, not just the file on disk), auto-continue without asking the user to retype `/build` — the manual-gate-stop is only there to handle context loss. Cold-start (new session, post-compaction, or the file on disk is the only evidence) keeps the manual gate at these two states only. The `phase-blocked` state never auto-continues regardless of session.

**Step 1 — Check `.build-state.json` in the project root.**

If present, read it and jump directly to the matching resume point:

| `step` value | Resume at |
|---|---|
| `constitution-complete` | Feature cycle → Step 1 (Spec), Phase 1 |
| `spec-complete` | Feature cycle → Step 2 (Frontend), phase from state file |
| `frontend-complete` | Feature cycle → Step 3 (Backend), phase from state file |
| `backend-complete` | Feature cycle → Step 4 (Review), phase from state file |
| `phase-complete` | **Resume-safety check first.** Read `dogfoodPid` from the state file. If null OR `kill -0 <pid>` fails, the dogfood handoff was never finished (almost always: compaction killed the original orchestrator before it could run). Run the full **`## Dogfood handoff`** section NOW, end-to-end (steps 1–7), printing the operator handoff block — before any wrap-up message. If `dogfoodPid` is non-null and alive, the server is already running: still run step 1 (commit + push — independently idempotent, the work might not yet be on origin even if the server is up), then print a one-line reminder of the URL + credentials + "what you can test" bullets, do not start a second server. Only then proceed to Mode 3 wrap-up for the completed phase, then the next feature cycle. |
| `phase-blocked` | Phase ended at the cap-hit "Stop" branch from `/review`. Surface the open issues, do **not** auto-resume. Wait for user to pick: another fix round, manual rollback, or accept-and-move-on. Do NOT run the dogfood handoff in this state. |
| `roadmap-complete` | "Roadmap complete — nothing left to build." Stop. |

**Backward compatibility — old enum values:** if you encounter `constitution-approved`, `spec-approved`, `frontend-approved`, `phase-approved`, or `complete`, treat them as the new `-complete` / `roadmap-complete` equivalents and rewrite the state file in place on the next gate. Do not error.

**Step 2 — If no `.build-state.json`, check `mission.md`.**

- **Missing** → New project path
- **Present** → Next feature path (Phase N+1)

---

## State file format

`.build-state.json` lives in the project root. Write it at every gate — overwrite the previous value.

Full schema (typed):

```ts
type BuildState = {
  /** Current or just-completed phase number from roadmap.md. */
  phase: number;

  /** Kebab-case feature slug from roadmap.md. */
  feature: string;

  /** Latest gate the orchestrator passed. See resume table above. */
  step:
    | "constitution-complete"
    | "spec-complete"
    | "frontend-complete"
    | "backend-complete"
    | "phase-complete"
    | "phase-blocked"
    | "roadmap-complete";

  /** /review auto-fix loop counter. Reset to 0 on convergence. */
  reviewIteration: number;

  /** sha256 of requirements.md captured at /spec Mode 2 user-approval reply.
   *  Downstream skills recompute on entry and surface drift if it changed.
   *  Empty string before the first /spec Mode 2 approval. */
  requirementsHash: string;

  /** Breadcrumb appended by the constituent skill on entry — e.g.
   *  "frontend.phase-2", "backend.phase-3", "review.step-1.5".
   *  /build owns step transitions; sub-skills only write currentSubStep. */
  currentSubStep: string | null;

  /** PID of the long-running dogfood server started after phase-complete.
   *  null when no dogfood server is up. /build kills the prior PID before
   *  starting a new dogfood server, and on the user's "stop dogfood" message. */
  dogfoodPid: number | null;

  /** Backend foundation build (design-independent plan.md groups built in the
   *  background while the user designs). Owner: /build. null = not started this
   *  phase. The branch is the truth source on resume — /backend skips any group
   *  whose verify script passes, regardless of this field. */
  foundationStatus: null | "running" | "done" | "failed";
};
```

Minimal example, freshly approved spec:

```json
{
  "phase": 1,
  "feature": "core-pipeline",
  "step": "spec-complete",
  "reviewIteration": 0,
  "requirementsHash": "9f2c1...",
  "currentSubStep": null,
  "dogfoodPid": null
}
```

**Write rules — one owner per transition:**

- **Sub-skills own the terminal `step` write on clean exit.** The terminal value is declared in each sub-skill's `## Invocation contract` section (canonical source — if `/build` and a sub-skill ever disagree, the contract wins). Sub-skills also write `currentSubStep` on entry and null it on clean exit.
- **`/build` only writes `step` on rollback.** When its post-sub-skill compliance check fails, `/build` rolls `step` back to the previous gate value and re-invokes the sub-skill. Compliance checks are belt-and-suspenders verification of what the sub-skill claimed to ship.
- **`/review` owns `reviewIteration`** — increment per loop, reset on convergence. See that skill's auto-fix loop policy.
- **`/spec` Mode 2 owns `requirementsHash`** — write it once on skeptic-panel convergence (auto-proceed); never overwrite outside Mode 2.
- **`/build` owns `dogfoodPid`** — set when starting the dogfood server, null on "stop dogfood" or before starting a fresh one.
- **`/build` owns `foundationStatus`** — set to `running` when spawning the background foundation agent, `done`/`failed` when it returns, null at each new phase's spec gate.

The cap-hit terminal writes from `/review` (`phase-complete` on user-Accept, `phase-blocked` on user-Stop) are sub-skill writes per the rule above — `/review` writes them after the user replies to the cap-hit binary.

Delete the file when `step` is set to `roadmap-complete` (roadmap finished).

---

## Model assignments

The main `/build` session runs at the user's default model (Opus, session default). Each sub-skill declares its model + mechanism in its own `## Invocation contract` section — those contracts are the canonical source. The summary below is for quick orientation only.

| Step | Model | Mechanism |
|---|---|---|
| `/bad-idea` Mode 2 (new-project idea pre-flight) | Opus (session default) | inline |
| `/ba` Mode 1 + 2 (new project, phase scope) | Opus (session default) | inline — spawns background research agent (Mode 2) |
| `/ba` Mode 3 (between-phase replan) | Sonnet | subagent |
| `/spec` Modes 1 + 3 | Sonnet | subagent |
| `/spec` Mode 2 (phase spec) | Sonnet + Opus drafters, Sonnet skeptics | **inline** — orchestrates parallel subagents |
| `/frontend` | Sonnet | subagent (internally subagent-free) |
| Backend foundation build (during user design wait) | session default | background agent — see Step 2 |
| `/backend` | Opus (session default) | inline — wave-dispatches Sonnet/Opus agents |
| Architectural review (inside `/backend`) | Opus | subagent — see `/backend` skill |
| `/review` | Opus (session default) orchestrating Sonnet subagents | **inline** — fleet, chunks, fix agents |

If this table ever drifts from a sub-skill's contract, the contract wins. Update the contract first, this table second.

**One level of subagents only.** Subagents cannot spawn subagents — any skill that orchestrates its own subagents runs inline in this session; any `subagent`-mechanism skill must be internally subagent-free. Full briefing/dispatch rules: `${CLAUDE_PLUGIN_ROOT}/skills/_shared/subagent-policy.md`.

**Sonnet subagent pattern** (every Sonnet callsite):
1. Write `currentSubStep: "<step>"` to `.build-state.json` — crash-recovery anchor.
2. Spawn `Agent(model: "sonnet")` with a prompt that says: `Read and execute ${CLAUDE_PLUGIN_ROOT}/skills/<sub-skill>/SKILL.md per its Invocation contract. Mode: <N if applicable>. Pass the inputs the contract names: <slot-fill from current /build context>.`
3. After the subagent returns: verify expected outputs (per the contract's "Outputs" column) exist. If missing, surface the error — do not proceed.
4. Null `currentSubStep` in `.build-state.json` (the sub-skill should have already nulled it on clean exit; this is belt-and-suspenders).

**`/ba` → `/spec` handoff:** when `/ba` runs inline (Modes 1 + 2), its decisions are already in context — pass them as the "BA decisions verbatim" input the `/spec` contract names. When `/ba` Mode 3 runs as a subagent, capture its return text and pass it verbatim as the "BA Mode 3 replan notes" input to `/spec` Mode 3.

**Special-case prompts.** Most callsites use the generic pattern above. Two exceptions where /build carries an extended prompt because the policy is hard-won:
- `/frontend` Stage 3 (the "do NOT skip Stage 2" warning + the Pencil-MCP-forbidden-during-Stage-3 rule) — see Step 2 below.
- `/backend` (no extended prompt — runs inline; the sub-skill's body is its own driver).

Everywhere else, /build's prompt is the generic line above.

---

## New project path

0. **`/bad-idea` Mode 2 — idea pre-flight** *(inline — Opus)*
   - Runs once at the very top of the new-project path. Never runs on the next-feature path or on any resume.
   - **Capture the idea.** If `/build` was invoked with an argument, that argument IS the idea brief — use it verbatim. If no argument, ask via `AskUserQuestion`: question = "What are you building? Give me the one-line idea so I can pressure-test it before we draft the constitution." (single text field, no preset options).

   - **Refinement loop — HARD CONTRACT (this is the most-violated step in the whole stack — do not skip, do not freelance).**

     Each iteration is exactly two actions in this order, with NO chat between them and NO action after the AUQ until the user replies:

     **(a) Invoke `/bad-idea` Mode 2 inline** with the current idea text. Mode 2 returns one round of analysis (short verdict + minimum-viable-shape — NOT the full lens table). Surface the returned prose to the user verbatim.

     **(b) Immediately issue `AskUserQuestion`** — header "Idea check", three options exactly:
       - **Proceed with this idea** — accept current shape, move to /ba Mode 1.
       - **Refine** — user pastes a response; fold it into the idea text as `<previous idea>\n\nUser refinement: <response>` and return to (a) for another iteration.
       - **Drop it** — abandon the project entirely.

     The AUQ in (b) is mandatory after every (a). It is illegal to:
     - Continue to /ba, /spec, or any state-file write before the user picks Proceed.
     - Skip the AUQ and instead chat with the user about the idea.
     - Treat a Refine response as conversation — it is loop input. Always fold it back into the idea text and re-invoke /bad-idea Mode 2.
     - Exit the loop on your own initiative because "the idea looks fine now" — only the user's Proceed or Drop selection exits.

     Loop bound: there is no max iteration count. Some users refine 1–2 rounds, some refine 5+. Keep looping until the user picks Proceed or Drop.

   - **On Proceed:** carry the final idea text forward as additional context for `/ba` Mode 1 — pass it inline at the top of the BA prompt as `Idea brief (post pre-flight): <final idea text>`. Continue to step 1.
   - **On Drop:** tell the user "OK, no project started." — exit /build. Do not write any state file. Do not call /ba.
   - Do NOT write `.build-state.json` during the pre-flight loop. The file is seeded for the first time only at the constitution-approved gate (step 3 below). The pre-flight is conversation-state only — if context is lost mid-pre-flight, the user re-invokes `/build [idea]` and starts the loop fresh.

1. **`/ba` Mode 1** — demand validation + constitution grill + master user flow *(inline — Opus)*
2. **`/spec` Mode 1** — write `mission.md` + `tech-stack.md` + `roadmap.md` + scaffold living docs.
   - Write `currentSubStep: "spec"` to `.build-state.json`.
   - Subagent prompt: `Read and execute ${CLAUDE_PLUGIN_ROOT}/skills/spec/SKILL.md. Mode: 1 (new project constitution). Project root: <absolute path>. BA decisions: <inline /ba decisions verbatim>.`
   - Verify `mission.md`, `tech-stack.md`, `roadmap.md` exist before proceeding. Capture the subagent's returned **product story** and **latent decisions** list.
3. **Gate: constitution approved (outcome-only — the user never reads the files).**
   - Surface the product story from the subagent's return: what the product does and for whom, what each phase delivers (one line each), what it will never do. Plain language — no file names, no tech stack. Technology is Claude's job; `tech-stack.md` exists but is never presented.
   - Ask any **experience-affecting** latent decisions from the subagent's list, one `AskUserQuestion` each ("I'd do X — it means Y for you. OK?"). Invisible-technical ones: record in `docs/decisions.md`, don't ask.
   - Then one `AskUserQuestion`: "This is the product we're building — approve and lock it in?" Options: **Approve** / **Adjust** (fold the change into the relevant constitution file via the same `/spec` Mode 1 subagent, re-surface, re-ask).
   - On Approve: write `.build-state.json`: `{ "phase": 1, "feature": "[phase-1-slug]", "step": "constitution-complete", "reviewIteration": 0, "requirementsHash": "", "currentSubStep": null, "dogfoodPid": null, "foundationStatus": null }`
   - Tell user: "Constitution set. Roadmap confirmed. **Run `/build` to start Phase 1.**"
   - Stop.

---

## Next feature path

Runs when `.build-state.json` has `step: "phase-complete"`, or when `mission.md` exists and no state file is present.

1. **Project state prime** (run first, always — see below)
2. **`/ba` Mode 3** — lightweight replan (what changed? roadmap still correct? what's next?)
   - Write `currentSubStep: "ba-replan"` to `.build-state.json`.
   - Subagent prompt: `Read and execute ${CLAUDE_PLUGIN_ROOT}/skills/ba/SKILL.md. Mode: 3 (between-phase replan). Project root: <absolute path>. Project state summary: <prime summary>.`
   - Capture subagent return text as `<ba-decisions>`.
3. **`/spec` Mode 3** — update living docs, run changelog, merge completed branch
   - Write `currentSubStep: "spec-wrap"` to `.build-state.json`.
   - Subagent prompt: `Read and execute ${CLAUDE_PLUGIN_ROOT}/skills/spec/SKILL.md. Mode: 3 (between-phase wrap). Project root: <absolute path>. BA replan output: <ba-decisions verbatim>. Project state summary: <prime summary>.`
   - Verify no pending conflicts in `roadmap.md` or `CHANGELOG.md` before proceeding.
4. Continue to **Feature cycle** for the next phase from `roadmap.md`.

### Project state prime

**Caching:** the prime summary is keyed on the current `git rev-parse HEAD` of the project repo. If you've already produced a prime summary earlier in *this same session* and the HEAD hasn't moved, skip the re-read and reuse the cached summary — surface "state primed (cached)" in working context. The HEAD changes when a phase merges, so a fresh prime always runs on the next-feature path after a real merge. Cross-session reuse is not safe — the cache is in-conversation only.

Before the BA replan starts (cold or post-cache-miss), re-ground on what exists. Read only this fixed set — do not re-read old `requirements.md` or `plan.md` files:

- `mission.md` — what the product is, who it's for, and the Master User Journey (Named Flows)
- `roadmap.md` — phase sequence; mark which phases are done vs. pending
- Last completed phase's `handover.md` (from the most recent `specs/YYYY-MM-DD-[feature]/` directory) — what actually shipped: screens, APIs, deviations
- `CHANGELOG.md` if it exists — one-line deltas per phase

Produce a 5–8 line working summary in context under `## Project state`:
```
Product: [one line from mission.md]
Flows: [Named Flows from mission.md Master User Journey — one line per flow, e.g. "Onboarding (4 steps), Core loop (3 steps)"]
Done: Phase 1 [slug] — [one-line what shipped], Phase 2 [slug] — [...]
Pending: Phase [N] [slug] — [one-line intent from roadmap], Phase [N+1] ...
Last phase deviations: [any flagged in last handover.md, else "none"]
```

This summary is read by `/ba` Mode 3 and carried through `/spec` Mode 3. Do not skip this step — cold-start invocations (context compacted, new session) depend on it to avoid heavy re-reads.

---

## Feature cycle

Same for every phase, whether Phase 1 of a new project or Phase N of an ongoing one.

### Step 1 — Spec

1. **`/ba` Mode 2** — phase scope grill, user stories, screen inventory, competitor research (background agent), **Outcome Card draft + user approval** *(inline — Opus)*. The card is the user's contract for the phase — the spec files are never user-approved.
2. **`/spec` Mode 2** — *(inline — orchestrates parallel subagents; one-level nesting rule)* writes `requirements.md` + `plan.md` + `validation.md` in `specs/YYYY-MM-DD-[feature]/`, validated by its skeptic panel against the approved card, auto-proceeds. Creates feature branch `phase-N-[feature-slug]`.
   - Write `currentSubStep: "spec"` and `foundationStatus: null` to `.build-state.json`.
   - Read `${CLAUDE_PLUGIN_ROOT}/skills/spec/SKILL.md` and execute Mode 2 inline in this session (it spawns drafter + skeptic subagents, which a subagent could not). Inputs per its contract: BA decisions verbatim, phase number, feature slug, approved `outcome-card.md` path.
   - Verify all three files exist and are non-empty before proceeding.
3. **Gate: spec complete (machine-approved).**
   - `/spec` Mode 2 already wrote `step: "spec-complete"` + `requirementsHash` on convergence. Re-verify; reset `reviewIteration` to 0 — a new phase starts a fresh review counter.
   - Tell user: "Specs written and adversarially validated against the card — [N] user stories, [N] screens, [N] API endpoints. Starting frontend design now."
   - The card is frozen — card changes restart Step 1.
   - **Auto-continue immediately to Step 2.** Do not stop.

### Step 2 — Frontend design

4. **`/frontend`** — reads `requirements.md` as contract → writes design brief → hands off to user to design in their chosen tool → user returns with approved design → writes `handover.md` as a frame index pointing backend at the design file (not a visual narration). Claude may use design-tool MCPs (Pencil, Figma, etc.) after Stage 3 to read the finished design and populate the frame index.
   - Write `currentSubStep: "frontend"` to `.build-state.json`.
   - Subagent prompt — use this template literally, do not paraphrase the phase contract or add encouragement to "skip ahead":
     ```
     Read and execute ${CLAUDE_PLUGIN_ROOT}/skills/frontend/SKILL.md.
     Project root: <absolute path>
     Phase: <N>
     Spec directory: <specs/YYYY-MM-DD-slug/>

     requirements.md is the contract — read it first.

     Execute the skill stages in order. Do not skip stages. Specifically:
     - Stage 2 (Design Brief) is unconditional. Write specs/<dir>/design-brief.md before any design work, regardless of whether a design file already exists in the project. An existing .pen / .fig / etc. is context, NOT a substitute for the brief — the brief captures phase-N intent that the design file does not.
     - Stage 3 External-tool path: hand off to the user and STOP. Do NOT call any mcp__pencil__* tool (or Figma/other-tool MCP equivalent) to design on the user's behalf. The user drives Stage 3 in their tool. Auto-mode does NOT authorize designing in Pencil — auto-mode only minimises questions, not user authorship of the design.
     - Pencil MCP is allowed only in Stage 5 (handover), and only to read the finished design (variables, frame IDs, screenshots for verification).

     Note on naming: "Stage" = /frontend's internal step (1–5). "Phase" = the project Phase from roadmap.md. We are working on project Phase <N>, executing all 5 of /frontend's Stages.

     The Stage 1 question (which design track for this phase?) is asked by /frontend at the start of every project phase — it overwrites mission.md `## Design Tool` with the picked value. Do not skip this question even if mission.md already has a value from a prior phase.
     ```
   - Verify `design-brief.md` exists before the user-design hand-off; verify `handover.md` exists before proceeding to the compliance check.

4b. **Backend foundation build — runs while the user designs (external tracks only).** The user's design pass is the longest wait in the pipeline; the design-independent half of the backend doesn't need it.
   - **Trigger:** the moment `/frontend` hits its external-track Stage 3 stop (brief delivered, user has gone off to design). Skip entirely on the `claude-code-impeccable` track (no user wait exists) or if plan.md has no `Design-dependent: no` groups.
   - **Spawn one background agent** (`Agent` tool, `run_in_background: true`, model omitted — inherits session default). Write `foundationStatus: "running"`. Brief:
     ```
     You are building the design-independent foundation of Phase <N>.
     Read <spec dir>/requirements.md, <spec dir>/plan.md, and tech-stack.md.
     Implement ONLY the plan.md groups marked `Design-dependent: no`, in dependency order.
     Discipline per ${CLAUDE_PLUGIN_ROOT}/skills/backend/SKILL.md Stage 2: Spec-Light header, verify-group-N.sh before
     implementation, tests-first for logic groups, root-cause iron law. NO design-token work, NO UI rendering,
     NO dev server. Commit per group on the current feature branch with a plain-English message.
     If blocked on a group after 3 hypotheses: skip it, leave it uncommitted-clean (stash nothing — revert
     partial work), continue with the next buildable group.
     Return: per-group status (done+committed / skipped+why), files changed, verify results. No commentary.
     ```
     Committing from the background agent is safe here — the main session is idle at the design stop and makes no commits until the user returns (exception to the no-agent-commits dispatch rule, by design; see `_shared/subagent-policy.md` Rule 6).
   - **When the user returns** with the design: collect the agent's result (await it if still running — tell the user "finishing the data layer I built while you designed"). Write `foundationStatus: "done"` (or `"failed"` if it errored — never block on it; `/backend` will build whatever is missing normally). Then continue `/frontend` Stages 4–5.
   - **Resume safety:** if context is lost during the design wait, the branch is the truth — foundation commits survive, and `/backend` Stage 2 skips any group whose verify script passes. A dead background agent with `foundationStatus: "running"` is treated as `"failed"` silently.

5. **Frontend compliance check** (run by `/build`, not `/frontend`) — **track-aware**:
   - Read `mission.md` `## Design Tool` to determine the track. Apply legacy-value mapping per `/frontend` backward-compat (`external` → `external-pencil`; `claude-code` / `claude-code-taste` → `claude-code-impeccable`).
   - Read `requirements.md` UI requirements (screen inventory). For each spec screen + state, the gate must find a corresponding artifact — what counts as "an artifact" depends on the track.
   - **Common to every track:**
     - Confirm `handover.md` exists.
     - Confirm `design-tokens.css` exists at the path `handover.md` references.
     - Confirm `handover.md` has a "Fonts required" section listing every font family the design uses.
   - **Track `external-pencil` / `external-other`:**
     - Confirm `handover.md` names a design file under "Design file — source of truth" and the file exists at that path.
     - For each spec screen + state: does the frame index have a row mapping it to a design node? Pass/fail per state.
   - **Track `claude-code-impeccable`:**
     - Confirm `specs/<this phase>/mockups/` exists and is non-empty.
     - Confirm `handover.md` has a mockup index — a section listing every mockup file under `mockups/` and the screens it covers.
     - For each spec screen + state: does the mockup index name a mockup file that covers it? Pass/fail per state.
     - The "Design file — source of truth" line is replaced by a "Mockups — source of truth" line pointing at the `mockups/` directory.
   - If any gaps: tell the user "Frontend handover incomplete: [list what's missing]. Auto-rolling back and re-running `/frontend`." Then **`/build` rolls `.build-state.json` `step` back to `spec-complete` and immediately re-invokes `/frontend`** (per the rollback rule in State file format above). Do NOT stop and wait for the user to retype `/build`. Loop until the compliance check passes.
   - If the required artifact for the track is missing entirely (no design file for external; no mockups directory for claude-code), apply the same auto-rollback + re-invoke — never stop and ask the user.

6. **Gate: frontend approved.**
   - Write `.build-state.json` with `step: "frontend-complete"`. Preserve all other fields.
   - Tell user: "Frontend complete — [N] screens designed and handed over. Starting backend now."
   - **Auto-continue immediately to Step 3.** Do not stop.

### Step 3 — Backend implementation

7. **`/backend`** — reads `requirements.md` + `plan.md` + `handover.md` → implements task groups in order using `code-harness` → architectural review (Opus) → integration testing

8. **Backend compliance check** (run by `/build`, not `/backend`):
   - Start dev server; poll until ready (max 30s)
   - For each API contract in `requirements.md`: send the specified request, verify response shape and status. Test each error condition listed.
   - If any contract fails: tell the user "Backend missing: [endpoint + what failed]. Auto-rolling back and re-running `/backend`." Then **`/build` rolls `.build-state.json` `step` back to `frontend-complete` and immediately re-invokes `/backend`** (per the rollback rule in State file format above). Do NOT stop and wait for the user to retype `/build`. Loop until every contract passes.

9. **Gate: backend complete.**
   - Write `.build-state.json` with `step: "backend-complete"`. Preserve all other fields.
   - Tell user: "Backend complete — [N] endpoints built and integration-tested. Starting review now."
   - **Auto-continue immediately to Step 4.** Do not stop.

### Step 4 — Review and approval

10. **`/review`** — reads `validation.md` → runs all automated checks → manual verification → UX dogfooding with the reviewer fleet → grades the outcome card. Only reached after both compliance checks pass.
    - Write `currentSubStep: "review"` to `.build-state.json`.
    - Read `${CLAUDE_PLUGIN_ROOT}/skills/review/SKILL.md` and execute it **inline in this session** (it orchestrates its own Sonnet subagents — fleet, walk chunks, report writer — and its fix agents must inherit this session's model; a subagent could do neither). Inputs per its contract: project root, phase, spec directory, validation file, outcome-card path.

11. **Gate: user approves phase. Phase-complete checklist — execute every sub-step in order. The handoff is the body of this gate, not an optional follow-up.**

    **a. Write state.** `.build-state.json` `step: "phase-complete"`. Reset `reviewIteration` to 0; preserve other fields.

    **b. Cap-hit Stop branch (early exit).** If `/review` ended on the cap-hit "Stop" branch (user picked Stop, not Accept): instead of phase-complete, write `step: "phase-blocked"`, surface the open issues, **skip the rest of this checklist**, wait for user instruction (another fix round, manual rollback, accept-and-move-on). Resume safety in the routing table does not run the handoff for `phase-blocked`.

    **c. Run the dogfood handoff — ALL of `## Dogfood handoff` steps 1 through 7.** This is not a forward reference and not optional — execute the section inline as the body of step 11c. It is what the operator actually needs at the end of a phase: the work pushed to origin, a running server, a LAN/Tailscale URL, dogfood credentials, and the "what you can test" bullets. **Do not** print "Phase complete" before this step finishes.

    **d. Print the phase-complete line.** *After* the handoff block from step c is on screen, add: "Phase [N] complete — [one-sentence summary]. **Run `/build` to update docs and start Phase [N+1].**"

    **e. Roadmap end check.** If this was the last phase in `roadmap.md`: write `step: "roadmap-complete"`, then delete the file.

    **f. Stop.**

    **Resume safety (compaction insurance).** If context dies between any two sub-steps above, a fresh orchestrator wakes up, reads state=`phase-complete`, and routes via the table — the routing row for `phase-complete` runs the same handoff if `dogfoodPid` is null/dead, so the operator still gets the URL+credentials they would have gotten in step 11c. The handoff is idempotent: if the server is already running, the routing path prints a one-line reminder instead of starting a duplicate. Compaction does not give the agent permission to skip the handoff.

---

## Dogfood handoff — body of `phase-complete` gate (also runs on resume)

This section is **not an optional follow-up.** It is the body of phase-complete step 11c, AND the resume action when the routing table sees `phase-complete` with no live `dogfoodPid`. Either way, every numbered step below executes end-to-end before the operator sees a "phase complete" message. Skip this entire section only if the gate is `phase-blocked`.

The handoff is **idempotent and re-entrant.** If `dogfoodPid` is non-null and `kill -0 <pid>` succeeds, the server is already running from a prior turn: print a one-line reminder of the URL + credentials + "what you can test" bullets and stop — do NOT start a duplicate server (BUT: still run step 1 below — commit + push is independently idempotent and the work might not yet be safely on the remote even if a server is up). If `dogfoodPid` is null or dead, run the full sequence below.

The dogfood server is separate from any one-shot test server `/review` may have spun up — keep that one disposable; this one is meant to live across sessions.

### 1 — Commit + push the phase work (always runs, idempotent)

Before any URL is printed, the phase's work must be safely on the remote. Operators dogfood from a different device (phone, Mac when the dev box is the PC, etc.); pushing first means the work survives a dev-box crash mid-dogfood and that the user can keep reviewing even if the local box goes offline.

This step always runs at the top of the handoff — even on resume, even if the dogfood server is already up. Both git ops are idempotent: `git status` quickly confirms a clean tree, and `git push` is a no-op if the branch is already up to date with origin.

1. **Verify branch.** Run `git rev-parse --abbrev-ref HEAD`. Expect the feature branch `phase-N-<feature-slug>` (matches `phase` + `feature` from `.build-state.json`). If on `main` or any other branch: stop and surface to the user — "Expected to be on phase-N-<slug> but on <actual>. Did the feature branch get merged or deleted prematurely?" Do not commit or push until the branch state is sorted.
2. **Stage and commit any uncommitted work.** Run `git status --porcelain`. If non-empty, the auto-fix loop or a late edit left work uncommitted. Stage exactly the files listed (never `git add -A` — see CLAUDE.md git safety) and commit with a message like:
   ```
   phase N complete: <one-line summary of what shipped, from the /review report>
   ```
   Use a HEREDOC with the standard `Co-Authored-By:` trailer naming the current session model (see CLAUDE.md committing protocol — never hardcode a version). If the staged set looks suspicious (contains `.env`, credential files, large binaries, or files outside `app/` and `specs/<this phase>/`), pause and ask the user before committing.
3. **Push the feature branch.** `git push -u origin <branch>`. If push is rejected (remote has commits the local doesn't), do NOT force-push — fetch, surface the divergence to the user, and wait for direction. If push is blocked by a hook or permission rule, surface the exact command to the user. If the project has no `origin` remote configured, skip the push silently and tell the user once: "No git remote configured — work is committed locally but not pushed. Consider `git remote add origin <url>` before next dogfood." (Don't fail the handoff over this — local commit is already enough recovery for most cases.)
4. **Confirm.** One short line back to the user: "Pushed phase N to origin/<branch> — N commits ahead of main." Then continue to step 2.

### 2 — Pick the run command

- If `docs/deployment.md` exists, follow whatever the project documents (it's the source of truth for any per-project quirks).
- Otherwise read `package.json` `scripts` and pick the dev script: prefer `dev` → `start` → `serve`. Run it via the project's package manager (`bun run dev` if `bun.lock` / `bun.lockb` is present, else `npm run dev`, else `pnpm dev` / `yarn dev` matching the lock file).
- If the project has no dev script, skip the handoff entirely and tell the user: "Phase complete. No dev script in package.json — handoff skipped." Do not invent one.

### 3 — Detect the LAN-reachable URL

Order of preference:
1. **Tailscale**: `tailscale ip -4 2>/dev/null | head -n 1` — if Tailscale is up and the operator owns this device on the tailnet, this is the most reliable inter-device IP (works across Mac, phone, anywhere on the tailnet).
2. **First non-loopback IPv4**: `hostname -I | awk '{print $1}'` — fallback for non-Tailscale setups.
3. Port: read from the dev script (e.g. `next dev -p 3000`); default to `3000` if not specified.

Final URL: `http://<IP>:<PORT>`.

### 4 — Seed dogfood credentials (only if the project has env-credential bootstrap)

Some projects support env-driven seed credentials (Phase 1 of A3 does — `EMAIL` + `PASSWORD_HASH` env vars seed a single user on first boot). Detect by grepping `app/src/**` (or the project's source tree) for one of: `process.env.EMAIL`, `process.env.PASSWORD_HASH`, or an explicit "seed user" pattern in init code. If detection finds **no** such pattern, skip this step — print the URL and skip the credentials line in step 7.

If detected:
- Seed password: literal `dogfood` (no special chars, easy to type on a phone).
- Hash with bcrypt: `bun -e 'import("bcrypt").then(b=>b.default.hash("dogfood",10).then(h=>console.log(h)))'` (or the equivalent `node -e` if bcrypt is in node_modules). The exact incantation depends on the project's bcrypt API — read the project's password-hashing util once and mirror its salt rounds.
- Email: `<project-basename>@<project-basename>.local` (e.g. `companion@companion.local`).
- Persist to `.env.dogfood` at the project root (gitignored). Cross-session: the file is reused on the next phase's handoff, so the password stays stable.
- Pick a stable DB path: `~/.config/<project-basename>/dogfood.db` (create the directory if missing). Set the project's DB-path env var (read the project's config util to find the variable name — common: `DB_PATH`, `DATABASE_URL`, `SQLITE_PATH`).

### 5 — Start the server in the background, capture PID

```bash
cd <project-root>
# Compose env from .env + .env.dogfood (so dogfood overrides DB path & seed creds)
env $(cat .env.dogfood 2>/dev/null | xargs) <run-command> > /tmp/<project>-dogfood.log 2>&1 &
echo $!
```

Capture the PID. Write it to `.build-state.json` as `dogfoodPid`. Poll the URL (max 30s) until it returns 2xx/3xx — then the server is ready.

### 6 — Generate the "What you can test" bullets

Read `outcome-card.md` for this phase — the bullets ARE the card: one bullet per primary outcome, using its "Success looks like" signal as the "what should happen" half. The user approved these exact promises at the start of the phase; now they verify them by hand. (Legacy phases without a card: fall back to `requirements.md` user stories.) Per bullet, in the operator's voice:

- Lead with the screen or feature name (so the operator knows where to look).
- One short sentence on what to do.
- One short sentence on what should happen — taken from the card's "Success looks like" line.

Example mapping (story → bullet):

> "As a user, I can add a workspace by picking a folder from `~/dev`, so the workspace appears in my list."
>
> → `Picker — click "+ Add workspace" — every folder in /home/tuana/dev shows up. Pick something to chat about it.`

Keep the list to one bullet per primary story. Skip secondary / negative-path stories — those are review territory, not dogfood territory.

### 7 — Print the handoff in operator voice

```
From your Mac (or phone), open:
  <URL>/<entry-route, default "/">
  Email: <email>
  Password: dogfood

---
What you can test
- <bullet 1>
- <bullet 2>
- <bullet 3>
- ...

When you want to stop the server: tell me "stop dogfood" and I'll kill it.
Or just leave it running and come back to it later.
```

Then continue with the standard `phase-complete` user-facing line: "Phase [N] complete — [one-sentence summary]. Run `/build` to update docs and start Phase [N+1]."

### Stopping the dogfood server

When the user says "stop dogfood" (or any clearly-equivalent phrase: "kill the dogfood", "shut it down", etc.), or before starting a new dogfood server in a later phase:

1. Read `.build-state.json` `dogfoodPid`.
2. If non-null, `kill <pid> 2>/dev/null` — then verify with `kill -0 <pid> 2>/dev/null` (exit 0 = still alive). If still alive after 2 seconds, escalate to `kill -9`.
3. Set `dogfoodPid` to null in `.build-state.json`.

If `dogfoodPid` is null but the user asked to stop: tell them "No dogfood server is running" — do not hunt for stray processes by name (risk of killing something the operator started themselves).

### Stale PID on resume

When `/build` resumes and reads a non-null `dogfoodPid`, verify it's still alive (`kill -0 <pid>`). If dead, null the field silently — do not announce. The dogfood server has died (reboot, OOM, manual kill); the next `phase-complete` will start a fresh one.

---

## Ground rules

- New project always runs `/bad-idea` Mode 2 pre-flight before `/ba` Mode 1. The pre-flight is in-conversation only — no state file write until the constitution is approved. Pre-flight does NOT run on the next-feature path or any resume.
- Constitution must exist before any feature work starts. `/build` on a new project runs Mode 1 first (after the idea pre-flight clears).
- The Outcome Card must be user-approved before any spec is written, and the spec files must pass `/spec`'s skeptic panel before implementation starts. Neither gate is skippable. The user never approves spec files — outcome gates only.
- The card is frozen after approval. Scope changes restart Step 1 — do not patch around the card or the specs.
- One level of subagents only — orchestrating skills run inline; see `_shared/subagent-policy.md`.
- Frontend compliance check must pass before backend starts.
- Backend compliance check must pass before `/review` starts.
- Living docs are updated at the start of the next `/build` invocation (Mode 3), not deferred past that.
- Project state prime runs before `/ba` Mode 3 on every next-feature entry — token-light re-grounding on mission, roadmap, and last handover. No exceptions on cold-start.
- Every gate writes `.build-state.json` — this is for crash recovery, not a signal to stop.
- Phase boundaries stop (`constitution-complete`, `phase-complete`, `phase-blocked`, `roadmap-complete`). Within-phase gates auto-continue (`spec-complete`→frontend, `frontend-complete`→backend, `backend-complete`→review).
- `phase-blocked` is the only terminal state where the next `/build` does NOT auto-continue silently — it surfaces the blocking issues and waits.
- Constituent skills must write their own `currentSubStep` breadcrumb on entry (e.g. `frontend.phase-2`). `/build` reads it on resume to know where the sub-skill stopped.
- Constituent skills also write their own terminal `step` on clean exit (compaction-safe handoff). `/build` re-writes the same value when it resumes or after a compliance check passes. Writes are idempotent. See "Write rules" in the State file format section for the full per-skill mapping.
- user approves scope, direction, and phase completion. Claude owns all technical decisions.
