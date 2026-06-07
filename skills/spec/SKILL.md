---
name: spec
description: >
  Translates BA session output into structured SDD documents. Three modes: (1) new project — writes mission.md + tech-stack.md + roadmap.md and scaffolds living docs; (2) start phase — parallel Sonnet+Opus drafters write requirements.md + plan.md + validation.md under specs/YYYY-MM-DD-[feature]/, validated against the user-approved outcome card by a 3-skeptic panel, then auto-proceeds (no user approval of spec files); (3) between phases — updates living docs, runs changelog, merges branch. Always invoked by /ba or /build after a drilling session. Never standalone.
---

# /spec — Structured Document Writing

Input comes from a `/ba` session. Output is structured markdown files. No drilling, no user questions — that belongs to `/ba`.

## Invocation contract

| Mode | Model | Mechanism | Inputs (caller passes) | Outputs (skill produces) | Terminal state (this skill writes) |
|---|---|---|---|---|---|
| 1 (new project) | Sonnet | subagent | BA Mode 1 decisions verbatim | mission.md, product.md, tech-stack.md, roadmap.md + scaffolded living docs; returns latent-decisions list + product-story summary for `/build`'s gate | `constitution-complete` |
| 2 (phase spec) | Sonnet + Opus drafters, Sonnet skeptic panel | **inline** (in `/build` session — orchestrates parallel subagents; one-level nesting rule, see `_shared/subagent-policy.md`) | BA Mode 2 decisions verbatim, phase number, feature slug, approved `outcome-card.md` path | requirements.md, plan.md, validation.md under `specs/YYYY-MM-DD-<slug>/`; feature branch created; **auto-proceeds — no user approval** | `spec-complete` (also writes `requirementsHash`) |
| 3 (between phases) | Sonnet | subagent | BA Mode 3 replan notes verbatim, project state summary | updated WIKI.md, README.md, docs/api.md, docs/decisions.md, docs/architecture.md, CHANGELOG.md; merged feature branch | none — `/build` owns the next-phase write; this skill only nulls `currentSubStep` on clean exit |

## Mode detection

| Context | Mode |
|---------|------|
| No `mission.md` in project root | Mode 1 — New project |
| `mission.md` exists, starting a phase | Mode 2 — Phase spec |
| Phase just completed | Mode 3 — Between phases |

---

## Wiki integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/_shared/wiki.md` with `$AGENT=spec` and `$TAGS` from `tech-stack.md` (Mode 1 has no stack — pass empty).

**Friction trigger:** the user modifies a spec after approval (scope creep signal). Title: `Phase <N> spec friction: <area>`. Body: which section changed, what the delta was, why the original spec missed it. Do not repeat for the same area within one session.

**Phase-wrap trigger:** at the end of Mode 3, after living docs are updated and before branch merge. One entry per phase. Title: `Phase <N> spec: <one-line summary>`. Body: what was non-obvious about this spec, what worked well, what should be sharper next time.

---

## Latent decisions — collection and routing (shared procedure — both Mode 1 and Mode 2)

Every spec write involves choices the user did not explicitly hand over. Some come from BA Mode 1/2 verbatim — those are *locked*, not latent. Latent decisions are the ones made silently while filling the templates because the BA pack didn't speak to them: a tech-stack micro-choice (which auth library), a scoping inference, a structural pattern, an interaction default (auto-save debounce), an exclusion added to keep the phase tight, an assumption about an unspecified flow. They are the most common source of post-spec rework.

The user never reads spec files, so latent decisions are not presented as a reading list. They are **collected by whoever wrote the docs and routed by whoever runs inline**:

**Collection.** Every doc-writing agent (Mode 1 subagent; Mode 2 drafters) returns a `## Latent decisions` list alongside its content — one sentence per choice that wasn't in the BA pack.

**What qualifies (include):**
- A scope inference not in the BA pack ("single-mart per pivot", "phase locks to read-only").
- A tech micro-choice not already in `tech-stack.md` ("Zod for validation, not Yup").
- A behavior default the user never named ("config auto-saves on drag with 800ms debounce").
- An exclusion added to "Not in this phase" because the BA pack was ambiguous.
- An assumption about an unspecified user flow ("dismissing the modal = cancel, not save").
- A name or label picked where the BA pack used a placeholder.

**What does NOT qualify (skip):** anything verbatim from the BA handoff; file names / directory structure / variable names; things the user already approved in a prior mode; trivial one-obvious-answer defaults. Aim for 2–6 items per writer; zero means you didn't look hard enough.

**Routing (run by the inline orchestrator — Mode 2 main session during reconciliation; `/build` for Mode 1):** merge and dedupe all writers' lists, then split each decision:

- **Experience-affecting** — the user would see, feel, or be constrained by it (an interaction default, a visible behavior, a capability boundary, anything touching the outcome card's promises): **ask immediately**, one `AskUserQuestion` per decision in the standard voice-rule shape — "I'd do X — it means Y for you. OK?" Fold the answer into the docs before the skeptic panel runs. Do not batch for a later gate.
- **Invisible-technical** — no user-perceivable difference between the alternatives: do NOT ask. Include the list in the skeptic panel's brief (lens 3 judges them better than a non-developer could) and record each in `docs/decisions.md` (`**[Decision]** — Why: [reason]. Alternatives: [considered and rejected].`).

When unsure which side a decision falls on, treat it as experience-affecting — a wasted 5-second question is cheaper than a phase of rework.

---

## Mode 1 — New Project Setup

**Step 0:** Run the Read wiki procedure from the Wiki integration section. Mode 1 has no `tech-stack.md` yet — invoke with `--agent spec` and no tags.

Input: constitution decisions from `/ba` Mode 1 (mission, tech stack, roadmap, exclusions).

**Approval flow (outcome-only — the user never reads the files):** Write all constitution files to disk (Steps 1–2), then return two things to `/build` in your final text: (1) the **Latent decisions** list per the shared collection procedure, and (2) a **product story** — a plain-language summary covering: what the product does and for whom (from mission.md), what each roadmap phase delivers in one line each, and what the product will never do. No file paths, no tech terms — `tech-stack.md` is written but never presented; technology is Claude's job. `/build` owns the gate: it surfaces the story, asks any experience-affecting latent decisions, and gets the user's approve via `AskUserQuestion`. This skill does not ask the user anything.

### Step 1 — Write constitution (4 files)

**`mission.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/mission.md`. Every field filled. Purpose: one crisp sentence. Target users: named and specific. Vision: one paragraph. Success: observable and specific.

**`product.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/product.md`. Every field filled. End-State Vision: one paragraph on what the finished product feels like to use. Screen Inventory: every screen in the finished product with phase label. Navigation Structure: flat map of how screens connect. Core Feature Surface: concrete actions a user can do at end-state. Named Flows: all flows from `/ba` Part C with phase labels on each step (3–6 flows). Phase 1 Skeleton Scope: every screen listed as live or stubbed — this is the instruction to `/frontend` to design the full IA in Phase 1.

**`tech-stack.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/tech-stack.md`. Every choice filled in. Constraints and non-negotiables listed explicitly (e.g. "strict TypeScript from commit 1", "all dependencies pinned exactly — no ^ or ~"). Explicit exclusions named with reasoning. Technical decisions table seeded with choices already made.

**`roadmap.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/roadmap.md`. Phases numbered and sequenced. Each phase: feature name + what it delivers + why it comes at that position. Global out-of-scope named explicitly.

### Step 2 — Scaffold living docs

Create these files with headers only (no placeholder content — an empty section with a header is fine, a section with fake content is not).

**Living-doc prefix rule:** every agent-facing living doc (`WIKI.md`, `docs/architecture.md`, `docs/api.md`, `docs/decisions.md`) starts with this exact line as its first line, before the title:

```
> Agent context — not for human reading.
```

The operator visually skips files that lead with this blockquote; agents still read them. `README.md` is the *only* doc here that targets humans, so it does NOT get the prefix.

- `README.md` — project name, mission (from mission.md), setup: "TBD", status: "Phase 0 — Constitution complete. Phase 1 not started." **No prefix.**
- `WIKI.md` — prefix line, then header: "# Project WIKI", subheader: "## Tech Stack Notes", body: "*(Added per phase)*"
- `CHANGELOG.md` — prefix line, then header: "# Changelog", body: "*(Auto-generated — do not edit manually)*"
- `docs/architecture.md` — prefix line, then tech stack section copied from tech-stack.md; rest TBD
- `docs/api.md` — prefix line, then header only: "# API Surface — *(Updated per phase)*"
- `docs/decisions.md` — prefix line, then seed with decisions already recorded in tech-stack.md Key Technical Decisions table

### Step 3 — Seed WIKI from global WIKI

If `~/.claude/wiki/` exists: search for entries tagged with the tech stack languages and frameworks. Copy relevant entries under `## From Global WIKI — [tech]` in the project WIKI.md.

If no global WIKI yet: skip silently.

### Output

**Checkpoint state file (compaction-safe handoff):** before output, write `.build-state.json` in the project root with:
- `phase: 1`
- `feature` set to the kebab-case slug of Phase 1 from `roadmap.md`
- `step: "constitution-complete"`
- `reviewIteration: 0`
- `requirementsHash: ""`
- `currentSubStep: null`
- `dogfoodPid: null`

This is the resume anchor for the new-project path. Constitution is a phase boundary — `/build` stops here and waits for the user to re-invoke. Without the state write, a compaction during the user's pause leaves the project looking unconstituted to the next `/build`, which restarts at Mode 1.

> **Constitution written.**
> **Files created:** mission.md, product.md, tech-stack.md, roadmap.md, README.md, WIKI.md, CHANGELOG.md, docs/architecture.md, docs/api.md, docs/decisions.md
> **Phase 1 feature:** [feature name from roadmap]
> **Ready to scope Phase 1.**

---

## Mode 2 — Phase Spec

**Step 0:** Run the Read wiki procedure from the Wiki integration section — pull tags from `tech-stack.md`, invoke with `--agent spec`. Summary goes into the skill's working context.

**Friction hook:** if during this phase the user changes the outcome card after approving it (scope creep signal), invoke the Write learning procedure with `--trigger friction --phase <N>` before re-entering Mode 2. Do not repeat for the same area within one session.

Input: phase decisions from `/ba` Mode 2 (scope, user stories, screen inventory, competitor patterns, API needs, constraints) + the user-approved `outcome-card.md` (the contract everything must trace to).

**Scope challenge (before writing anything):** Search the existing codebase before speccing new work.

1. For each feature in the BA handoff: does any existing file, route, component, or utility already handle part of it? List what exists and what's genuinely new.
2. Identify the minimum set of changes needed — flag if more than 8 files or 2+ new abstractions are involved and ask whether to reduce scope before writing the spec.
3. Note any deferred items in `roadmap.md` that overlap with this phase — can any be bundled without expanding scope?

Output a brief "What already exists / What we're actually adding" summary at the top of the working context before writing. This is for the spec author only — not a user-facing deliverable.

**Pipeline (no user approval — the card is the contract):** Draft all three spec files via parallel agents (Step 2) → main-session reconciliation + latent-decision routing (Step 3) → skeptic panel + fix loop (Step 4) → write files and auto-proceed (Step 5). The user approved the outcome card in `/ba`; the specs are machine-validated against it. The only user contact in this mode is the immediate-ask on experience-affecting latent decisions (see shared routing procedure) and the rare skeptic-cap-hit surface.

**Hash on auto-proceed (drift detector):** the moment Step 4 converges and the files are written:

1. Compute `sha256sum specs/YYYY-MM-DD-[feature-slug]/requirements.md | cut -d' ' -f1`.

2. Write `.build-state.json` with:
   - `requirementsHash` set to the computed value
   - `step` set to `"spec-complete"`
   - `phase` from `roadmap.md` (the phase being specced)
   - `feature` set to the kebab-case feature slug
   - `reviewIteration` reset to 0 (fresh phase, fresh review counter)
   - `currentSubStep` nulled
   - `dogfoodPid` preserved if present

This is the resume anchor + drift detector in one write. If context is compacted between this skill ending and `/frontend` starting, the next `/build` reads `spec-complete` and jumps straight to Step 2 (Frontend) instead of re-running `/ba` Mode 2 + the 3-doc spec write.

The hash is taken **after** the skeptic panel converges and the final files are on disk, never mid-fix-loop — an in-progress edit must not generate a stale hash. Only `/spec` Mode 2 writes `requirementsHash`.

### Drift detection (downstream skills should run on entry)

`/frontend`, `/backend`, and `/review` recompute the hash on entry and compare to the recorded `requirementsHash`. If different:

- Surface to the user once: "requirements.md changed since spec approval. The downstream work is now building against a different contract. Continue, or restart spec?"
- Do not auto-block — the user may have intentionally tweaked a typo. Just make the drift loud.
- If user picks "Continue": clear `requirementsHash` (set to empty string) so the same warning doesn't fire repeatedly.

### Step 1 — Determine spec directory

Use today's date: `date +%Y-%m-%d`. Feature slug from the roadmap entry, kebab-case (e.g. `user-auth`, `product-listing`). Create: `specs/YYYY-MM-DD-[feature-slug]/`.

Also create a feature branch: `git checkout -b phase-N-[feature-slug]` where N is the phase number from roadmap.md.

### Step 2 — Parallel spec drafting

Spawn two agents in parallel (`Agent` tool — briefing rules in `${CLAUDE_PLUGIN_ROOT}/skills/_shared/subagent-policy.md`). Both work from the BA Mode 2 decisions verbatim, the approved outcome card, and the scope-challenge summary from Step 1.

**Sonnet agent (`model: "sonnet"`) — user-facing specs (requirements.md + validation.md):**

```
You are writing the user-facing spec files for a software phase. Do not write any files — return content only.

BA handoff: [paste full BA Mode 2 decisions verbatim]
Outcome card (the user-approved contract — every requirement must serve it): [paste outcome-card.md verbatim]
Scope summary: [paste scope challenge output — what already exists / what's new]
Phase: [N], Feature slug: [slug], Spec directory: specs/YYYY-MM-DD-[slug]/
Tech stack: [paste tech-stack.md constraints and non-negotiables]

Write requirements.md using schema at ${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/requirements.md. Fill every section:
- Frontmatter block: phase (integer), type (initial | feature | rebuild)
- Scope: one paragraph on what this phase delivers
- User stories: one per major user action, each referencing its Named Flow + step from product.md, format [Flow name, Step N]
- UI requirements: every screen + every state (default, empty, loading, error, mobile) — behavior and key elements only, no visual treatment
- Data model: all tables/schemas, field names, types, relationships
- API contracts: every endpoint — method, path, auth, request shape, success response, ALL domain-specific error conditions (not just 500)
- Constraints & context: business rules, tone, patterns from tech-stack.md
- Excluded from this phase: named explicitly

Write validation.md using schema at ${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/validation.md:
- Automated checks: specific commands that exit 0 on success (TypeScript typecheck, named unit tests, curl for each API contract)
- Manual verification: specific steps at named viewport sizes, binary pass/fail
- Every user story must have at least one manual check
- Outcome checks: one binary, demonstrable check per outcome-card primary outcome — phrased so a non-technical person can verify it on screen
- Definition of Done: all criteria listed; the outcome checks are part of it

Return both files as raw markdown content, each labelled with its filename, followed by a "## Latent decisions" list — one sentence per choice you made that wasn't in the BA handoff or outcome card (scope inferences, behavior defaults, added exclusions, assumed flows). No other commentary.
```

**Opus agent (`model: "opus"` — alias only, never a version-pinned ID) — technical implementation plan (plan.md):**

```
You are writing the technical implementation plan for a software phase. Do not write any files — return content only.

BA handoff: [paste full BA Mode 2 decisions verbatim]
Outcome card (the user-approved contract — every task group must serve it): [paste outcome-card.md verbatim]
Scope summary: [paste scope challenge output — what already exists / what's new]
Phase: [N], Feature slug: [slug]
Tech stack: [paste tech-stack.md — include constraints, non-negotiables, pinned versions, excluded patterns]
Existing codebase relevant context: [paste key file paths, route structure, component names for files this phase touches]

Write plan.md using schema at ${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/plan.md. Organize as numbered task groups, each independently verifiable:
- Describe what each group delivers (user-visible capability, data path, API surface, integration point)
- Name components, files, modules, endpoints specifically
- Each group: one-line Verify command + Depends on field + Design-dependent field (`yes` if the group renders UI or needs design tokens/handover; `no` if it's pure data layer, API routes, or business logic buildable from requirements.md alone — these groups can be built before the design exists)
- Design-agnostic: no hex values, Tailwind classes, pixel sizes, font names — those come from the design file at build time
- Sub-tasks specific enough to implement without ambiguity
- Typical sequence: primitives/layout shell → page composition → data layer → API routes → integration/edge cases → story walk
- Each group should be implementable and verifiable independently before the next starts

Return plan.md as raw markdown content, followed by a "## Latent decisions" list — one sentence per choice you made that wasn't in the BA handoff or outcome card (tech micro-choices, structural patterns, scoping inferences). No other commentary.
```

### Step 3 — Main-session reconciliation review + latent-decision routing

Receive both agents' outputs. Check all six of the following. Resolve gaps inline.

1. **Story → task coverage:** every user story in requirements.md maps to ≥1 task group in plan.md. Add a task group if missing.
2. **Task → validation coverage:** every task group in plan.md has ≥1 corresponding check in validation.md. Add the check if missing.
3. **API contract alignment:** every API endpoint in requirements.md appears in plan.md (as a group or sub-task to implement) and in validation.md (as an automated curl test). Add whichever is missing.
4. **Data model consistency:** the data model in requirements.md is consistent with schema/table structure implied in plan.md. Reconcile any field-name or type mismatches.
5. **Outcome coverage:** every outcome-card primary outcome has ≥1 user story serving it and one Outcome check in validation.md. Add what's missing.
6. **Scope alignment:** plan.md groups implement only what requirements.md scopes, and requirements.md scopes only what serves the card — no extras, no gaps. Trim or add as needed.

Then run the **routing half of the shared latent-decisions procedure** on the drafters' returned `## Latent decisions` lists (the drafters made these choices, not you — do not re-derive from your own context): merge, dedupe, ask experience-affecting ones via `AskUserQuestion` NOW (fold answers into the docs), route invisible-technical ones into the Step 4 panel briefs and `docs/decisions.md`.

Only surface anything else to the user if there is a structural conflict requiring a product decision (e.g. plan.md assumes a Named Flow that product.md doesn't define).

### Step 4 — Skeptic panel + fix loop (replaces user approval)

The specs are validated adversarially, not by the user. Spawn **three Sonnet skeptics in parallel** (`Agent` tool, `model: "sonnet"`, context-isolated briefs — paste the docs in; no file access, no fix attempts). Each brief ends with: `Return findings ONLY in this format, one per line: [Critical|Major|Minor] <doc>: <what is wrong> → <specific fix>. If a lens finds nothing, return "CLEAN: <lens name>". No commentary.`

1. **Completeness/error-paths skeptic.** Gets: requirements.md, outcome card. Attack: missing states (empty/loading/error/mobile), unhandled error conditions, API contracts with vague errors, "what happens when…" holes (double-submit, expired session, concurrent edit, zero data, huge data), undefined transitions.
2. **Testability skeptic.** Gets: requirements.md, validation.md, outcome card. Attack: stories too vague to verify, validation checks that can't actually fail or don't prove the story, automated checks that aren't runnable commands, outcome checks a non-technical person couldn't verify on screen, Definition-of-Done gaps.
3. **Outcome-traceability + reality skeptic.** Gets: all three docs, outcome card, Step-1 scope-challenge summary, the invisible-technical latent decisions. Attack: requirements serving no card outcome (scope creep — name it for cutting), card outcomes under-served by the specs, contradictions with constitution docs or the existing codebase, latent decisions that are actually wrong calls.

**Fix loop:** collect findings. Fix every Critical and Major in the docs (you authored them in Step 3 — these are doc edits, no code). Minors: fix if one-line, otherwise drop. Re-run **only the lenses that had findings**, with **fresh** skeptics (subagent-policy Rule 7). Cap: 3 rounds. Converged (no Critical/Major) → Step 5. Cap hit with Critical findings still open → surface once to the user in outcome language only ("The plan for Phase N has a hole I can't close alone: [plain-language issue]. [Option A / Option B]?") via `AskUserQuestion` — never show the findings list raw.

### Step 5 — Write files and auto-proceed

Write the three reconciled, panel-validated spec files:
- `specs/YYYY-MM-DD-[slug]/requirements.md`
- `specs/YYYY-MM-DD-[slug]/plan.md`
- `specs/YYYY-MM-DD-[slug]/validation.md`

(`outcome-card.md` is already in the directory from `/ba`.) Then run the **Hash on auto-proceed** write from the top of this mode and continue straight to frontend. No user gate.

### Output

> **Phase [N] spec written and adversarially validated.**
> **Directory:** `specs/YYYY-MM-DD-[feature]/`
> **Branch:** `phase-N-[feature-slug]`
> **Spec:** [N] user stories, [N] screens, [N] API contracts, [N] plan groups, [N] validation checks
> **Skeptic panel:** converged in [N] round(s) — [N] findings fixed
> **Ready for frontend design.**

---

## Mode 3 — Between-Phase Docs Update

**Step 0:** Run the Read wiki procedure from the Wiki integration section — pull tags from `tech-stack.md`, invoke with `--agent spec`.

**Phase-wrap hook:** after Step 6 (changelog) and before Step 7 (merge branch), run the Write learning procedure with `--trigger phase-wrap --phase <N>`. Exactly one entry per phase — draw the summary from the spec files and git log of the just-completed phase.

Input: replan notes from `/ba` Mode 3. Run this in the same session the phase completed — not later.

### Step 1 — Update README.md

Change status line to: "Phase N complete — [feature]. Phase N+1 next: [feature from roadmap]."

### Step 2 — Update WIKI.md

Add a new section: `## Phase N — [Feature Name] Learnings`

Write what was learned, not what was done. Git log has the what. WIKI has the why and the surprise. Minimum 3 entries if anything non-obvious happened. Each entry: topic + what was learned (useful to a future agent starting fresh). Examples: gotchas, tech stack quirks, patterns that worked, patterns that failed.

### Step 3 — Update docs/api.md

Add or update every endpoint that changed this phase. Keep it current — this is the live API surface, not the spec (which is frozen after approval).

### Step 4 — Update docs/decisions.md

Add any non-obvious technical choices made during implementation. Format: `**[Decision]** — Why: [reason]. Alternatives: [what was considered and rejected].`

### Step 5 — Update docs/architecture.md

If new components were added or the data model changed, update the relevant sections.

### Step 6 — Run changelog

Generate `CHANGELOG.md` by running:

```bash
git log --format="%ad|%s" --date=short | awk -F'|' '
  {
    if ($1 != date) { date=$1; print "\n## " date }
    print "- " $2
  }
' >> CHANGELOG.md
```

Or if the project has a `scripts/changelog.py`: `python3 scripts/changelog.py`

Review the output — remove merge commits and branch housekeeping. Keep only meaningful changes.

### Step 7 — Merge branch

```bash
git checkout main
git merge phase-N-[feature-slug] --no-ff -m "Phase N complete: [feature name]"
git branch -d phase-N-[feature-slug]
```

### Output

> **Docs updated and branch merged.**
> **WIKI:** [N] learnings added for Phase N
> **CHANGELOG:** updated through [date]
> **Branch:** `phase-N-[feature-slug]` merged to main
> **Next:** Phase N+1 — [feature from roadmap]

---

## Ground rules

- **The user approves outcomes, never spec files.** Mode 1's gate is `/build`'s product story; Mode 2's gate is the outcome card approved in `/ba`. Spec files are machine-validated (Mode 2 skeptic panel) and auto-proceed. The only mid-mode user contact is the immediate-ask on experience-affecting latent decisions and the rare cap-hit surface — both via `AskUserQuestion`, both in plain outcome language.
- Subagent rules (nesting, briefing, models, output contracts) live in `${CLAUDE_PLUGIN_ROOT}/skills/_shared/subagent-policy.md` — Mode 2 orchestrates subagents and therefore runs inline in the `/build` session.
- Every field in every schema must be filled. Empty sections are failures — write them or remove the heading.
- API contracts must include all error conditions, not just success responses.
- Every user story must have a corresponding validation check.
- Docs update (Mode 3) happens in the same session as phase completion — not deferred.
- WIKI records what was learned, not what was done.
