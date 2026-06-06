---
name: backend
description: >
  Backend implementation skill for SDD projects. Primary input: requirements.md (what to build) + plan.md (task groups) + handover.md (frontend context). Implements plan.md task groups in order using code-harness, then runs architectural review (Opus) and integration testing against every API contract in requirements.md. Trigger on: /backend, or when /build reaches the backend step.
---

# /backend — Backend Implementation

Input: find the most recent `specs/YYYY-MM-DD-[feature]/` directory and read:
1. `requirements.md` — what to build (the contract)
2. `plan.md` — task groups to implement in order
3. `handover.md` — what frontend built and expects from backend

If these files are missing: stop. "No phase spec found. Run `/ba`, `/spec`, and `/frontend` for this phase first."

## Invocation contract

| Mode | Model | Mechanism | Inputs (caller passes) | Outputs (skill produces) | Terminal state (this skill writes) |
|---|---|---|---|---|---|
| (single mode) | Opus | inline (in `/build` session) | spec directory; reads requirements.md + plan.md + handover.md + design-tokens.css + design file/mockups | implemented backend per plan.md task groups; integration-tested per requirements.md API contracts | `backend-complete` |

Inline-Opus because the work is deep architectural reasoning + code-harness discipline + Stage 3 adversarial review. The architectural review in Stage 3 spawns its own Opus subagent loaded with `/adversarial-review`.

**Naming note: "Stage" vs "Phase".** The numbered Stages below (Stage 0–4) are this skill's *internal* steps for one project phase. The project Phase (Phase 1, Phase 2, … from `roadmap.md` and `requirements.md` frontmatter) is the higher-level slice the whole `/build` pipeline is working on. When you see `Phase <N>` in placeholders (wiki entry titles, adversarial review prompts, completion summary) that refers to the project Phase. When you see `Stage <N>` it refers to the section in this skill.

---

## Voice rules

The user is not a developer. Plain language throughout — no file paths, function names, or stack traces in summaries.
- Surface only decisions that affect what user experiences or what the product offers. Everything else is yours.
- When a technical fork affects user-visible behavior: "I'd do X — it means Y for users. OK?"
- Never ask the user to run a server, manage a process, or open a browser. Agent owns all of that.
- Escalation is a last resort: try the primary approach, then a meaningfully different one, before surfacing to user.

---

## Wiki integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/_shared/wiki.md` with `$AGENT=backend` and `$TAGS` from `tech-stack.md`.

**Friction trigger:** in Stage 3, for each Critical or Worth-considering finding from `/adversarial-review` that you act on. One entry per acted-on finding. Skip declined findings and nits. Title: `Phase <N> backend friction: <issue name>` (the placeholder `<N>` is the project Phase, not the Stage). Body: what was flagged, severity, how it was fixed, what would have prevented it.

**Phase-wrap trigger:** once after Stage 4 integration testing passes, before Completion. Title: `Phase <N> backend: <one-line summary>`. Body: which schemas/patterns surprised, which approach worked vs. dropped, what to watch next time.

---

## Stage 0 — Read and confirm scope

**Step 0:** Run Read wiki (see Wiki integration) before reading spec files.

1. Read `requirements.md` — **start with the frontmatter block** (`phase`, `type`). The `type` field governs how this phase is implemented (see "Phase-type rules" below). Then read the contract, data model, and constraints.
2. Read `plan.md`: understand the task groups and their sequence. Plan.md is design-agnostic by contract — do not expect visual details here.
3. Read `handover.md`: understand what the frontend built and what API shape it expects. Note any deviations from requirements.md — these must be resolved before implementation starts.
4. Check `tech-stack.md` for non-negotiables (e.g. strict TypeScript, no ORM, exact version pinning).
5. **Open the design file** if `handover.md` lists one under "Design file — source of truth". See Design file rules below — this step is non-negotiable for any phase that touches UI.
6. **Import design tokens** — if `handover.md` references a `design-tokens.css`, import it from the app's global stylesheet (e.g. `@import './design-tokens.css'` at the top of `globals.css`). Tokens are the single source of truth for colors, fonts, sizes. Do not re-extract by hand, do not approximate.

Log internally: "Building: [API endpoints]. Phase type: [initial/feature/rebuild]. Constraints: [relevant tech-stack constraints]. Out of scope: UI, design." Proceed immediately — the spec was already user-approved.

---

## Phase-type rules

The `type` field in `requirements.md` frontmatter changes how you implement. Read it before anything else.

### `initial`
Greenfield build. No existing UI patterns to preserve. Follow the design file as the sole visual authority. Code-harness conventions ("follow existing codebase patterns") apply *only after* you've built the first working version — there is no prior codebase to follow.

### `feature`
Additive work on an existing product. Follow existing codebase patterns (file layout, naming, component style, state management). The design file describes only the *new* surface area; existing screens are untouched unless the design explicitly updates them.

### `rebuild`
Visual or structural redesign of existing product. **Existing UI patterns are explicitly overridden by the design file.** Do not preserve components, layouts, or chrome from prior phases just because they exist. If the design shows a top-header pattern and the codebase has a left sidebar, delete the sidebar — do not restyle it. Code-harness's "no invented abstractions" still applies to utilities, stores, and logic; it does **not** apply to visual layout or component structure.

If the `type` field is missing: read `mission.md`. If `mission.md` indicates a **greenfield** project (no prior shipped phases — `roadmap.md` shows Phase 1 still in progress, no `CHANGELOG.md` entries), stop and ask the user since the choice between `initial` and `rebuild` is non-obvious. Otherwise default to `feature` — this is the steady-state case and asking every phase is friction. Log the default in working context: "type defaulted to feature (mission.md shows N shipped phases)".

If the `type` field is present but unclear (typo, unknown value), stop and ask: "Phase type `<value>` not recognised — is this `initial`, `feature`, or `rebuild`?"

---

## Design file rules (UI phases only)

The design file is the **visual and structural source of truth** — it supersedes `design-brief.md`, any textual description in `handover.md`, and any visual pattern the existing codebase already uses. The brief describes intent; the design file is what the user approved and is happy with.

**Before any UI group in Stage 2:**

1. Open the design file at the path in `handover.md` "Design file" section.
   - Pencil: `mcp__pencil__open_document` → `mcp__pencil__get_editor_state` to list frames → `mcp__pencil__batch_get` with the frame IDs from the handover frame index to read structure → `mcp__pencil__get_screenshot` to visually verify the rendered frame.
   - Figma: use the configured MCP or open the exported asset bundle referenced in handover.
   - Other: follow the "How to read it" instructions in handover.
2. For each state the group implements, fetch the corresponding frame/node via the handover frame index.
3. **Mirror the design tree, not the existing codebase.** If the design shows a top-bar-only layout but the existing code has a left sidebar, remove the sidebar — do not restyle it. Structural patterns (navigation, page chrome, modals vs. sheets, spacing scale) come from the design file.
4. **Reusable components in the design = reusable components in code.** If the design defines a "Phase Card / Running" symbol, implement it as one component with state variants, not as inline markup duplicated per screen.
5. **Visual tokens come from the design file.** Exact colors, padding, radius, type sizes, weights — extract from the design nodes. Do not invent, do not approximate from the brief, do not retain legacy tokens "because they were close enough."
6. If the handover's frame index is missing entries for states listed in `requirements.md` UI Requirements, stop: "Frame index incomplete for [state]. Return to `/frontend` to fill it." Do not improvise the visual.
7. If the design file and `requirements.md` disagree on whether a state exists: design file wins on visuals; requirements.md wins on behavior. If the conflict is structural (a screen is in one and not the other), surface to the user.

**During each UI group:**

- Keep the relevant design frame open/accessible while implementing. Re-screenshot at the end of the group and compare against the design frame before self-verify.
- When a layout question comes up ("should this be a drawer or a modal?", "what's the spacing here?"), answer it by reading the design node — do not guess.

---

## Stage 1 — Test plan

**Codepath diagram.** Before writing any implementation code, map every function and code path that this phase will add or change. For each: list the happy path, the edge cases (null input, empty input, boundary values), and the error paths (upstream failure, invalid state). Mark each path `[TESTED]` or `[GAP]`. This diagram is the test plan for Stage 2 — use it during Stage 2 to decide which groups need tests and what edge cases to cover.

**Regression coverage is mandatory.** Any code path that previously had a passing test and is touched by this phase requires a regression test. The regression test must fail without the fix and pass with it.

For pure-UI / pure-refactor / pure-visual-rebuild phases with no new logic: skip the codepath diagram. The regression-test rule still applies if you touch any tested code path.

---

## Stage 2 — Implementation (group by group)

Implement `plan.md` task groups in order using the `code-harness` skill for every group. Each group is independently verifiable before moving to the next.

For each group:
1. Apply **Spec-Light**: post `TASK: [group description] / ESTIMATE: [time] / VERIFY: [command]`
2. Write `verify-group-N.sh` before touching implementation
3. For groups with new logic: write tests before implementation and confirm they fail first. Skip for pure-UI / pure-config groups. Regression tests for any previously-tested code paths this group touches are mandatory — write them first too.
4. Implement — minimum code to pass the tests / deliver the slice
5. Self-verify: run `verify-group-N.sh`
6. If red: apply root cause discipline before touching anything:
   - **Iron law:** no fix without root cause first. Trace the failing codepath — read what's actually happening, check what recently changed.
   - Form one hypothesis. Test it with a targeted check or temporary log. Do not write fix code before verifying the hypothesis.
   - After 3 failed hypotheses: stop. Surface to user: "Stuck after 3 approaches. Tried: [list]. Suspect: [what]. Need: [what decision]."
   - Red flag — stop immediately if each fix reveals a new problem: you're fixing at the wrong layer.
   - **WTF score** — track across this task group: start 0%, add 15% per revert, 5% per fix touching >3 files, 20% for any change outside this group's scope. Above 20%: stop and surface before continuing. Hard stop at 3 complete reverts.
   If 2× estimate with no root cause found: surface to user in plain language.
7. **Visual compliance gate (UI groups only).** See below — this is a mandatory stop before commit for any group that renders UI.
8. Commit with a plain-English summary user can read in 2 seconds

### Visual compliance gate

A UI group is any group that renders to the DOM (a component, a page, a layout change). Before commit:

1. **Start the dev server** if not already up. Poll until it responds.
2. **For every frame this group implements** (cross-ref with `handover.md` frame index):
   - Capture the rendered state via browser automation (Puppeteer, `browse` skill, or MCP equivalent). Use the right viewport (1280px desktop; 390px mobile variants).
   - Capture the design frame via `mcp__pencil__get_screenshot({ nodeId })` (or tool equivalent).
   - Post both screenshots side-by-side to the user with a one-line asymmetry report: what matches, what doesn't.
3. **Explicit asymmetry check** — look for and flag:
   - Presence of elements not in the design (e.g. a sidebar that shouldn't be there)
   - Absence of elements that are in the design (e.g. a CTA pill missing from a top header)
   - Wrong layout chrome (modal vs. sheet, sidebar vs. top-bar, centered vs. left-aligned)
   - Wrong palette (using legacy colors instead of design tokens)
   - Wrong typography (using legacy fonts instead of those in `design-tokens.css`)
4. **Gate:** if the built screen is clearly structurally different from the design frame, **do not commit**. Fix the drift, re-verify. If you cannot resolve the drift in two attempts, surface to the user with both screenshots: "Built differs from design on [specific points]. Direction?"

This gate is the single most important check in the pipeline. Skipping it defeats the entire "design file is source of truth" contract.

For every new codepath:
- Name every error condition — no unnamed catch-alls
- Happy path + 3 shadow paths: null input, empty input, upstream error
- Observability: log anything that can fail silently
- Follow patterns from `tech-stack.md` and existing codebase — no invented abstractions

---

## Stage 3 — Architectural Review (`/adversarial-review`)

After all groups are implemented, run a single Opus advisor pass loading `${CLAUDE_PLUGIN_ROOT}/skills/adversarial-review/SKILL.md`. The seven lenses (module depth, abstraction necessity, data-flow legibility, seam placement, information hiding vs relabeling, logic consolidation, naming honesty) cover the full structural sweep — there is no separate "general architecture" pass.

Subagent prompt (Opus):

```
Read and execute ${CLAUDE_PLUGIN_ROOT}/skills/adversarial-review/SKILL.md.

Target: Phase <N> backend implementation in <project root>.
Specifically the files touched by this phase's plan.md task groups under
the following directories: <list app/src/** and any other backend dirs touched>.

Spec context: <spec dir>/requirements.md, <spec dir>/plan.md.

Apply ALL seven lenses. Produce the consulting report in the format the skill specifies — named challenge, specific revision, severity tag per finding. Cite specific files and line ranges. Do NOT paste file contents.

Critical = will cause concrete pain in the next 1–2 phases.
Worth-considering = adds long-term friction.
Nit = stylistic.

Empty report (no findings, no clean-lens trace) = malformed run. At minimum, list which lenses came back clean.
```

When the report comes back, surface it inline (so the user sees the consulting findings) under a `### Adversarial review report — Phase <N>` heading.

### Revision round

Act on the report:
- **Critical findings**: address all of them. For each, post a one-liner: "Acted on `<finding title>` — `<what changed in plain language>`."
- **Worth-considering findings**: address the ones whose revision is cheap (single-file change, no contract impact). Decline others — for each declined, post: "Declined `<finding title>` — reason: `<one sentence>`."
- **Nits**: ignore unless the fix is trivial (one-line change). For any acted-on nits, post a one-liner.

The revision round is the only behavior change. The report itself does not block — Stage 4 (integration testing) runs whether or not every finding is addressed. Anything not acted on is a tacit accept; future phases won't re-litigate it unless new evidence surfaces.

If the revision round touches code that had passing tests, write a regression test before the change (per Stage 1 regression-coverage rule).

Once the revision round writes are complete, log a one-line summary in working context: `Adversarial review: <N> critical → <N> acted, <N> worth-considering → <N> acted / <N> declined, <N> nits → <N> acted.` This goes into the Stage 4 / completion log, not chatted to the user.

---

## Stage 4 — Integration Testing

Test the feature as a whole against every API contract in `requirements.md`. Not per-task — the full feature end-to-end.

1. Start dev server; poll until ready (max 30s)
2. For every API contract in `requirements.md`:
   - Send the exact request specified
   - Verify response shape matches spec exactly (field names, types, status codes)
   - Verify each error response: send a request that triggers each error condition, verify the correct status and message
   - Verify edge cases: empty/null input, concurrent requests where relevant
3. Cross-check against `handover.md`: does the implementation match what frontend expects? Flag any shape mismatches.
4. On failure (attempt 1): diagnose and fix. On failure (attempt 2): try a different diagnosis. Still failing → surface to user: "Endpoint [X] is failing. Tried: [approach 1, approach 2]. Suspect: [what]. Need: [what decision from you]."

---

## Completion

**Pre-Completion:** Run the phase-wrap Write learning (see Wiki integration) — one entry per phase.

**Checkpoint state file (compaction-safe handoff):** before invoking `/review`, write `.build-state.json` with `step: "backend-complete"`. Preserve every other field (`phase`, `feature`, `reviewIteration`, `requirementsHash`, `dogfoodPid`); null `currentSubStep`. This is the resume anchor — if context is compacted between this skill ending and `/review` starting, the next `/build` reads `backend-complete` and jumps straight to Step 4 (Review).

> **Backend complete.**
> **Built:** [what the backend provides — one sentence]
> **API endpoints:** [count] implemented, all integration-tested
> **Status:** Ready / Has known issues / Blocked
> **What I'd watch:** [one line on anything fragile — only if relevant]

Immediately invoke `/review`. Do not stop or wait for the user.

---

## Ground rules

- requirements.md is the contract. Build what it specifies — nothing more.
- plan.md task groups are the implementation sequence. Do not reorder or skip groups.
- Code-harness gates every group. No exceptions.
- tech-stack.md non-negotiables apply from the first line of code.
- Opus reviews architecture, not logic. Code-harness handled logic. Stage 3 runs a single Opus pass via `/adversarial-review` (seven structural lenses). The report does not block but triggers a revision round (Critical = act, Worth-considering = act-if-cheap, Nit = ignore).
- Agent owns servers, processes, and test runners. Never ask user.
- user is the last resort, not the first.
