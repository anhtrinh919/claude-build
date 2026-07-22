---
name: build-spec
description: >
  The SDD spec authority — it drills the user live AND authors the docs, merging BA + spec into one skill. Explicit mode arg: constitution / phase / replan. **constitution** — reads the locked Product Shape + research.md, runs Steps 1.4-1.6 of the master flow: 1.4 a second, deeper grill-me interview (mission + tech constraints), 1.5 constitution writing (Master User Journey, authors mission.md/product.md/tech-stack.md via Opus drafters), 1.6 roadmap (drill + draft + author roadmap.md + patch product.md's phase labels). **phase** — runs per-phase feature research, drills scope + stories + screens, owns the Outcome Card gate (the only user approval at spec time), then two parallel Opus drafters write requirements/plan/validation, drift-reviewed against the card. **replan** — updates living docs, runs the changelog, merges the branch. Runs inline; orchestrates Opus drafter + Sonnet research leaf agents. Owns the Outcome Card gate. Invoked by /build.
---

> **Part of `/build`.** On a resume with an active `.build-state.json`, enter through the `/build` orchestrator — it routes here; don't drive this skill off the state file directly (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`).

# /build-spec — Drill + Author

One skill does both halves of the spec: it **drills the user** for every decision the build needs, then **authors the structured docs** those decisions imply. You ask; leaf agents draft; you write. No separate BA handoff — the drilling session's decisions flow straight into authoring in the same context.

Mechanism is **inline** always: you hold live user gates (drilling, the Outcome Card) *and* orchestrate leaf agents, and both force inline (`_shared/subagent-policy.md` Rule 1). The only subagents anywhere are the leaf workers you fan out — Opus drafters and one Sonnet research agent.

> **Judgment, not lookup.** The mode bodies below are ladders and templates, not scripts. Push deeper when answers are thin; stop when a topic is settled. Sense evasion, scope-creep, and "success" described as features instead of pain. A quoted string is *intent to convey in plain language*, never a script to recite — unless marked **exact**. Never a file path, schema name, or stack term to the user (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Mode | Reads | Authors | Leaf agents | Terminal `step` |
|---|---|---|---|---|
| **constitution** (Steps 1.4-1.6) | locked **Product Shape** + `research.md` (from build-shape) | `mission.md` · `product.md` · `tech-stack.md` · `roadmap.md` + living-docs scaffold | Opus drafters (core docs) | **none** — `step` rides on build-shape's `shape-complete` the whole time, `currentSubStep` tracks 1.4/1.5/1.6; you return the product story + latent decisions; the **orchestrator** writes `constitution-complete` after its user boundary gate |
| **phase** | constitution docs · `roadmap.md` · `backlog.md` | user-approved `outcome-card.md` + `specs/YYYY-MM-DD-[feature]/{requirements,plan,validation}.md` + `requirementsHash` | 1 Sonnet feature-research agent + 2 Opus drafters | **`spec-complete`** — you write it on clean exit (also writes `requirementsHash`) |
| **replan** | just-completed phase spec + `roadmap.md` | living docs reconciled to as-built · `CHANGELOG.md` · merged branch | none | none — orchestrator owns the next-phase write |

Model **Opus** (you and the drafters). Feature-research leaf **Sonnet**. State file: **`.build-state.json`**. Mode is passed explicitly by the orchestrator (`constitution` / `phase` / `replan`) — do not auto-detect; act on the arg.

## The two research jobs (read before anything — R1)

There are **two distinct research jobs** in this stack; do not conflate them:

- **Product research** — ONE time, in **build-shape**, written to `research.md`. Covers the whole product's market, demand, and comparable products at the idea level.
- **Feature research** — EVERY phase, HERE in **phase** mode, before drilling. Covers how comparable products handle *this phase's specific feature set* (what screens exist, which patterns are standard, what's distinctive).

`research.md` does **not** cover per-phase scope. Never skip feature research on the assumption that the shape-stage product research already answered it — it did not. Feature research runs every single phase.

## Division of labor (internalize before drilling)

**User decides:** what to build, who it's for, scope, priorities, what success looks like — **and any technical call the user will feel** (UX or performance).
**You decide:** how to build it — framework, file structure, API shape, library choices, test strategy, naming. The invisible-plumbing layer, where no choice produces a felt difference.

When a technical choice produces a **felt** difference — how a flow works, or how fast it runs — it is the user's call: surface it as a **fork** (the genuine options + each one's plain-language tradeoff + a recommended default), in outcome language, never the jargon. The user picks; you never pre-decide a felt call and ask them to rubber-stamp it.

**Looks like a product question, is actually invisible plumbing** — decide silently *when the choice itself is unfelt*: auth provider, database engine, deployment target, state management, test framework, CSS strategy, cache/queue/worker. But when one produces a felt consequence (cache nightly = instant but up to a day stale, vs. live = fresh but slower), that *consequence* is a fork — surface the felt tradeoff, not the technology. Criterion + fork form: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`.

## Drilling discipline (constitution + phase)

Apply `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/drilling-discipline.md` at full depth — every drilling session in this mode (the Step 1.4 product interview, the phase-mode scope grill) runs at this discipline, no exceptions.

## Latent decisions — collection + routing (constitution + phase)

Latent decisions are choices made silently while authoring because the drilling didn't speak to them: a tech micro-choice, a scope inference, a behavior default, an assumption about an unspecified flow.

**Collection.** Every drafter returns a `## Latent decisions` list — one sentence per choice not covered by the drilling session or the outcome card. **Include:** scope inferences, tech micro-choices not in `tech-stack.md`, unnamed behavior defaults, "not this phase" exclusions from ambiguity, flow assumptions, resolved placeholder names. **Skip:** anything verbatim from the drilling; file/variable names; already-approved items; trivial one-answer defaults. Aim 2–6; zero means you didn't look hard enough. For any item the user will feel, the drafter carries its **fork** (options + one-line tradeoff each + recommended pick) so you can surface it without re-deriving.

**Routing** (you, after merging + deduping the drafters' lists — route *their* choices, don't re-derive):

- **`felt-impact`** (user sees it, feels it, waits on it, or is constrained by it) → surface as a **fork** via one `AskUserQuestion` per decision (batch closely-related ones); fold the answer into the docs before you write. Record the resolved fork in `docs/decisions.md ## User decisions`.
- **`invisible-plumbing`** (no felt difference either way) → do NOT ask; record in `docs/decisions.md ## Technical decisions` (`[Decision] — Why: … Alternatives: … rejected`).

When unsure, treat as **`felt-impact`** — a 5-second fork is cheaper than a phase of rework. Before surfacing any fork, check `docs/decisions.md ## User decisions` first — if already settled, honor it, never re-litigate.

## Brain integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md` with `$AGENT=spec` and `$TAGS` from `tech-stack.md` (constitution has no stack — pass empty). Read at the first substantive step. **Friction trigger:** the user re-answers/rejects the same decision topic 3+ times, or modifies a spec after approval — one entry per topic per session. **Phase-wrap trigger:** once per phase at the end of replan, before merging the branch.

---

## Mode: constitution

**Read first — the shape is settled.** build-shape passes a locked **Product Shape** (what it is / who it's for / the core job / the forks already chosen — single vs. multi, the spine, the boundary / what's kept out of the core shape) and `research.md`. These big-shape decisions are **settled**, and **demand was already validated at the shape gate** — do NOT re-drill demand or re-open the shape. Confirm each shape element in passing ("Building this for just you, one shop — right?") and drill the *details* within it. If a drill answer genuinely contradicts the shape, surface it as a shape change and confirm — don't silently re-derive.

### Step 1.4 — Grill-me product interview

Write `currentSubStep: "spec.1.4"` (`step` stays `shape-complete` — build-shape already wrote it; only `currentSubStep` moves through this mode).

A second, deeper trunk/branch/leaf tree-walk — run at full Drilling discipline depth (tree-completeness, not a count) — covering Mission and Tech-stack constraints only; roadmap is its own step (1.6).

- **Mission:** what does this do for the user (one sentence — outcome, not features)? What does it NOT do? What experience should it create (tone/feeling — one paragraph)?
- **Tech stack:** any language/framework constraints, infra/hosting requirements, or technologies explicitly ruled out? Only surface *genuine* business constraints or existing infrastructure — you decide the rest silently.

No gate — auto-move to 1.5, transition announcement ("Mission and tech constraints settled. Writing the constitution next.").

### Step 1.5 — Constitution writing

Write `currentSubStep: "spec.1.5"`. No gate — auto-move. Two labeled subsections, each kept individually legible, run in order:

#### Master User Journey

Map the full product journey across three layers. Becomes `## Master User Journey` in `mission.md` — the end-to-end anchor every phase's user stories cite.

- **Layer 1 — Core Jobs (JTBD):** per actor, 1–2 statements "When [situation], I need to [goal], so I can [outcome]." 3–5 total. Motivation-level only — no feature names.
- **Layer 2 — Named Flows:** 3–6 named flows, each 3–5 steps at verb-noun granularity. Example — **Onboarding:** Register → Connect data → Configure → First run. Draft from the conversation, then one `AskUserQuestion` (complete / missing something / needs reordering); adjust once.
- **Layer 3 — Flow-to-phase mapping:** label each step with the roadmap phase that delivers it, e.g. `Register (Ph1) → Connect data (Ph1) → Configure (Ph2)`. A gap signals a missing phase — flag it as a note to resolve at Step 1.6, don't block here (the roadmap doesn't exist yet).

#### Author + scaffold

Spawn Opus drafter leaf agent(s) with a context-isolated brief — the Product Shape, `research.md`, the drilling decisions, and the schema paths — to author `mission.md` and `tech-stack.md` in full, plus `product.md` **except** its Screen Inventory phase-label column and App Map phase-coloring (leave both as `TBD — patched at Step 1.6`; phase numbers don't exist until the roadmap is drafted). Do NOT author `roadmap.md` here — that's Step 1.6. They return raw markdown; you write to disk. (Core docs are Opus-authored, never Sonnet — judged on completeness, not brevity.) Brief ends with **containment: no spawning agents, no /build*, no `claude -p`, no commit, no server, don't address the user — return content/paths only.**

**Core docs** (schemas under `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/`, every field filled except the two roadmap-dependent ones noted above):
- **`mission.md`** — purpose (one crisp sentence), named/specific target users, one-paragraph vision, observable success, `## Master User Journey`.
- **`product.md`** — End-State Vision; Screen Inventory (every screen, phase-label column `TBD` for now); Navigation Structure (flat map); **App Map** (one Mermaid `flowchart` — screens as nodes, user actions as edges; phase-coloring `TBD` for now — the single human-facing diagram in the stack, regenerated from the two sections above, never hand-maintained); Core Feature Surface; Named Flows (phase-label `TBD`); Phase 0 Foundation Scope (every screen to final visual polish, static, mock data — the instruction to build-design to build the full IA to polished static in Phase 0).
- **`tech-stack.md`** — every choice filled; constraints/non-negotiables explicit (e.g. "strict TypeScript from commit 1", "all deps pinned exactly — no `^`/`~`"); exclusions named with reasoning; Key Technical Decisions table seeded. `tech-stack.md` is the **widest-read doc** in the stack — every phase skill reads it as the constraints authority.

**Scaffold living docs** per `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/living-docs.md` (headers only — an empty section is fine, fake content is not): `CLAUDE.md` (index + `## Project directives`, no prefix), `backlog.md` (empty buckets, no prefix), `README.md` (mission + status "constitution in progress, roadmap not yet drafted", no prefix), `CHANGELOG.md`, `docs/architecture.md`, `docs/api.md`, `docs/decisions.md`. Every agent-facing living doc (`docs/architecture.md`, `docs/api.md`, `docs/decisions.md`) opens with the exact first line `> Agent context — not for human reading.`; `README.md` and `CLAUDE.md` do NOT. Seed `docs/decisions.md ## User decisions` with the forks already resolved (the Product Shape's chosen forks + any felt-impact calls answered during the grill), each as `[Decision] — Chose: X. Why: … Options were: … (Phase 0 · date)`; seed `## Technical decisions` from the tech-stack Key Technical Decisions table. None of this scaffolding depends on `roadmap.md`.

Transition announcement into 1.6.

### Step 1.6 — Roadmap

Write `currentSubStep: "spec.1.6"`.

**The user drives the slicing — the most important step to get right.** Full discipline: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/roadmap.md`. Apply `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/roadmap-axis.md` for the slicing axis doctrine.

Drill, then collaborate: ask for the full feature set roughly (list, don't scope) → draft Phase 0 + one feature per phase ordered by value + dependency → surface the drafted list via `AskUserQuestion` (confirm order / reorder / resize / change which comes first), loop until the user is happy → ask what is globally out of scope. If the Master User Journey's Layer 3 flagged a flow-to-phase gap at Step 1.5, resolve it here — it usually means a missing phase.

**Author `roadmap.md`** — Phase 0 (Foundation) first, then one vertical slice per phase; carry the user's slice sequence verbatim; each phase names feature + what it delivers + why it's at that position; global out-of-scope named. Author `roadmap.md` with Phase 0 fully specified and phases 1+ marked deferred with the reason if the shape isn't concrete enough yet, then sequence them at the Phase 0 wrap.

**Patch `product.md`** — fill in the Screen Inventory's phase-label column and regenerate the App Map's phase-coloring now that phases exist. This is a mechanical edit against the doc authored at Step 1.5, not a re-author.

**State checkpoint:** write `.build-state.json` with `phase: 1`, `feature` (kebab slug of Phase 0 from roadmap), `reviewIteration: 0`, `requirementsHash: ""`, `currentSubStep: null`, `dogfoodPid: null` — `step` stays `shape-complete` (this mode never writes its own terminal step; see below). This is the resume anchor — without it a compaction during the user's pause restarts at Step 1.4.

### Return (do NOT write the terminal step)

Return to the orchestrator: (1) the **latent-decisions list** (felt-impact ones carrying their fork), and (2) a **product story** — what the product does and for whom, what each roadmap phase delivers in one line, what it will never do. No file paths, no tech terms. Note that `product.md` has an App Map the user can open in any markdown viewer — do NOT paste raw Mermaid into the return text. The orchestrator owns the constitution boundary gate (story + felt-impact forks + approval) and writes `constitution-complete` after it. You ask the user nothing at the end of this mode.

---

## Mode: phase

Input: the constitution docs, `roadmap.md`, `backlog.md`, and the phase/feature the orchestrator names. Read `mission.md`, `tech-stack.md`, `roadmap.md` before asking anything — never start from zero on an existing project, and never ask the grill a question those docs (or the existing codebase) already answer.

### Step 0 — Feature research (R1 — every phase, spawn first, wait for it)

Spawn **1 Sonnet leaf agent** (`Agent`, `run_in_background: false`) with a context-isolated brief: the feature name, the product one-liner from `mission.md`, and the goal — find 2–3 apps solving *this specific feature* via `WebSearch` and extract what screens exist, which patterns are standard, what's distinctive. Brief ends with **containment: no spawning agents, no /build*, no `claude -p`, no commit, no server, don't address the user — return content/paths only.** Wait for it to return before asking the user anything below — the grill in A–B must be grounded in what it found, not reconciled against it afterward. This is per-phase research and is distinct from shape-stage product research (R1) — it runs regardless of what `research.md` covered.

Once it returns: present the comparison, then one `AskUserQuestion` — what's worth stealing or avoiding: *use some of these patterns / avoid all / take note but design differently.* Carry the answer into A–B below (reference the specific patterns when framing scope/story questions, don't ask blind and reconcile later).

**Backlog triage (before the first question):** if `backlog.md` exists, read it. Flat tasks (`T-N`) that fit this phase without expanding the card → note by ID during orientation ("I also noticed T-3 — could fold it in here if you want"). Open dogfood reports (`DF-N`) → name by ID and ask whether to fix this phase or separately. Do NOT spend an `AskUserQuestion` on backlog items; surface by ID naturally. Always reference by ID.

### Fork-first scoping (governs A–B)

Every decision this phase that the user will *feel* — how a flow works, what a screen does, how fast it responds — is the user's call, surfaced as a **fork** while drilling, not baked into the card to rubber-stamp at the end. Whenever a real either/or appears (live-filter vs. apply-button; show-all vs. paginate; inline-edit vs. modal; instant-save vs. explicit-save), put the genuine options side by side with each one's tradeoff and a recommended default, and let the user pick *before* it hardens into the card. The card then records decisions the user already made, not picks awaiting discovery at approval.

### A. Scope grill

This is the design-concept / mental-model grill for the phase — run it at full Drilling discipline depth (tree-completeness, not a count).

- **What this phase delivers:** quote the roadmap entry verbatim and ask if it still matches. What can the user do at the end they couldn't before? What's explicitly out of scope even if related? Scope was set at constitution — confirm and detail, don't re-scope. If the roadmap entry feels wrong, that's a constitution change — surface it, ask whether to update `roadmap.md` first.
- **Who and why:** who uses this feature specifically? What's the consequence if it doesn't exist or doesn't work? What does "this works" look like from their perspective?

### B. User stories + screen inventory

- **User stories:** per major action, "As [actor], I can [specific action] so that [specific outcome]." Push for specificity — "As a manager, I can filter the product list by supplier so that I see only their items" is a story; "I can manage the list" is not.
- **Screen inventory:** per story — what does the user see? Empty state? Loading? Error? Mobile? Edge cases (double-submit, session expired, back button mid-action)? Format: Screen / State / Key UI elements / Primary user action.
- **Business rules + constraints:** rules governing this feature? Patterns to follow from the existing codebase (check `tech-stack.md`)? Tone/language constraints?

### Primary flow

Identify the 1–3 user stories whose failure would make the feature useless — these become the stop criteria in build-review. Stories not listed are secondary (bugs block but don't stop the review).

```
Primary flow:
1. [Story: exact wording from Part B]
2. [Story]  ← optional
3. [Story]  ← optional
```

### Outcome Card gate (the only user approval at spec time)

Spec files are machine-validated downstream and never shown to the user — the card is their contract.

1. **Draft** from the drilling session using `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/outcome-card.md`. Primary outcomes map **1:1** to the Primary flow stories (same count, same order). Everything in user language — no endpoints, schemas, or component names. The card records the forks the user already picked during scoping, not new decisions to discover here. Leave `approved` unset.
2. **Write** to `specs/YYYY-MM-DD-[feature-slug]/outcome-card.md` (create the directory now; today's date from `date +%Y-%m-%d`).
3. **Evaluate Ceremony scope** (below) and fold its recommendation into the same approval question.
4. **Surface verbatim in chat**, then one `AskUserQuestion` to approve — convey: this card is the contract for Phase N; everything built and reviewed traces back to it. Fork: **Approve — build directly** (recommended default when Ceremony scope says narrow) / **Approve — full spec + design** (recommended default when it says full) / **Adjust** (fold in changes, re-surface, re-ask — loop until Approve).
5. On Approve: set `approved: <today>` and `ceremony: full|narrow` (from the choice made) in frontmatter. A later card change restarts this mode.

### Ceremony scope (evaluate before presenting the card)

Most phases don't need the full ceremony. A genuinely narrow phase can be built directly and dogfooded — skipping spec-authoring and design ceremony entirely — for real token and time savings. Recommend **narrow** only when every check below holds; any one false → recommend **full** (tie-break: default to the safer, fuller path).

| Check | Narrow requires |
|---|---|
| New screens | zero — every screen this phase touches already exists to polished static from a prior phase (Phase 0 built the full IA; most later phases only wire behavior into it) |
| Primary flow | exactly 1 story |
| Mechanism | no Behavioral-mechanism-diagram trigger (no async/background jobs, real-time sync, multi-actor flow, state machine, role/permission model) |
| API surface | ≤3 endpoints, all simple CRUD |

State the verdict plainly when you present the card, in one line of *why* (e.g. "this phase only wires the existing list screen to real data — one story, no new screens — I'd build this directly rather than write full specs").

### Scope challenge (before authoring)

Search the existing codebase before speccing new work. For each feature: does any existing file, route, component, or utility already handle part of it? List what exists vs. what's genuinely new. Identify the minimum set of changes — flag if more than 8 files or 2+ new abstractions are involved, and ask whether to reduce scope first. Note any deferred `roadmap.md` items that overlap and could bundle without expanding scope. Output a brief "What already exists / What we're actually adding" summary into your working context (author-only, not user-facing).

### Author

Create the feature branch `phase-N-[feature-slug]` (N from `roadmap.md`) either way.

**Full ceremony — two parallel Opus drafters.** Spawn **two Opus drafters in parallel** — both work from the drilling decisions verbatim, the approved card, the scope summary, and `tech-stack.md`. The split is for parallel speed + the divergence-reconciliation it surfaces, not a model split (both Opus — spec docs are core docs). Read the brief file immediately before spawning and fill its bracketed slots:
- **Requirements drafter** — `${CLAUDE_PLUGIN_ROOT}/skills/build-spec/briefs/drafter-requirements.md` → `requirements.md` + `validation.md`.
- **Plan drafter** — `${CLAUDE_PLUGIN_ROOT}/skills/build-spec/briefs/drafter-plan.md` → `plan.md`.

Each brief ends with the containment line and instructs the drafter to return raw markdown + a `## Latent decisions` list, writing no files. Then run Reconciliation and Drift review below.

**Narrow ceremony — you author directly.** No leaf-agent spawn, no reconciliation, no drift review. Write `requirements.md` + `plan.md` yourself, in one pass, to the same schemas the drafters would otherwise produce — every field still filled; narrow means less process around the docs, not thinner docs. Do **not** write `validation.md`. Skip straight to Write + auto-proceed.

### Reconciliation — the 8 checks (full ceremony only)

Receive both outputs. Check all eight; resolve gaps inline (doc edits):

1. **Story → task coverage** — every user story in requirements.md maps to ≥1 task group in plan.md. Add a group if missing.
2. **Task → validation coverage** — every plan group has ≥1 check in validation.md. Add it if missing.
3. **API contract alignment** — every endpoint in requirements.md appears in plan.md (as a group/sub-task) and in validation.md (as an automated curl test). Add whichever is missing.
4. **Data model consistency** — the requirements.md data model matches the schema/tables implied in plan.md. Reconcile field-name/type mismatches.
5. **Outcome coverage** — every outcome-card primary outcome has ≥1 story serving it and one Outcome check in validation.md.
6. **Scope alignment** — plan.md implements only what requirements.md scopes, and requirements.md scopes only what serves the card. No extras, no gaps.
7. **Endpoint ↔ screen binding (no orphans, no data-less screens)** — every endpoint names ≥1 consuming screen in its `Consumed by` field (or is marked `internal`), and every screen whose UI implies data needs has a backing endpoint. An endpoint no screen calls is scope creep — flag for cutting; a data-needing screen with no endpoint is a missing contract — add it.
8. **Screen reachability (no dead-ends)** — trace the Navigation Structure in product.md across this phase's screens; every screen must be reachable from home AND able to return, directly or via a named flow. A screen with inbound paths but no way back is a dead-end — add the missing transition to requirements.md.

Then run the **routing half of latent-decisions** on the drafters' returned lists (their calls — don't re-derive): merge, dedupe, surface `felt-impact` ones as forks via `AskUserQuestion` NOW (fold answers in), carry `invisible-plumbing` ones to the drift review and record in `docs/decisions.md`. Surface to the user only a structural conflict needing a product decision (e.g. plan.md assumes a Named Flow product.md doesn't define).

**Behavioral-mechanism diagram (conditional — most phases skip).** Scan plan.md for a mechanism whose *behavior over time* isn't inferable from screens + endpoints: async/background jobs, real-time sync, multi-actor flows, state machines, role/permission models. If one exists, write a single Mermaid `sequenceDiagram` into `docs/decisions.md` under the prose decision it illustrates. Pure CRUD = no diagram.

### Drift review — 3 lenses (full ceremony only)

You hold all three docs + the card in context; every fix is an inline doc edit. Resolve every issue under:

1. **Drift from the card (headline).** Any requirement/screen/plan-group serving no card outcome → cut it. Any card outcome under-served or missed → add what's needed. Any contradiction with the constitution or existing codebase → reconcile. Re-judge the `invisible-plumbing` latent decisions; fix wrong calls, promote any with felt impact to a fork.
2. **Completeness / error-paths.** Missing states (empty / loading / error / mobile), unhandled errors, vague API errors, "what happens when…" holes (double-submit, expired session, concurrent edit, zero data, huge data). Specifically verify the **implicit states** the card leaves unsaid: optional fields valid as null or absent; array fields valid as empty `[]` (not just null); whitespace-only required strings treated as invalid (trim before the empty check); GET list endpoints returning `200 + { items: [], total: 0 }` when the user has zero records (never `404`); every paginated endpoint with an explicit max `pageSize` cap stated as a number (not just "paginated").
3. **Testability.** Stories too vague to verify, validation checks that can't fail or don't prove the story, automated checks that aren't runnable commands, outcome checks a non-technical person couldn't verify on screen, Definition-of-Done gaps.

Fix everything by editing the docs. Surface to the user only a hole you cannot close alone (e.g. the card is internally contradictory): one `AskUserQuestion` in outcome language — never a raw findings list.

### Write + auto-proceed

**Full ceremony:** write the three reconciled, drift-reviewed files under `specs/YYYY-MM-DD-[slug]/`: `requirements.md`, `plan.md`, `validation.md` (`outcome-card.md` is already there).

**Narrow ceremony:** write `requirements.md` + `plan.md` only, under the same spec dir — no `validation.md`.

Either way, then:

1. `sha256sum specs/YYYY-MM-DD-[slug]/requirements.md | cut -d' ' -f1` — the hash is taken **after** drift review (full) or immediately after authoring (narrow), never mid-review. Only phase mode writes `requirementsHash`.
2. Write `.build-state.json`: `requirementsHash` (computed), `step: "spec-complete"`, `phase` + `feature` (from roadmap), `reviewIteration: 0`, `currentSubStep: null`, `dogfoodPid` preserved, `stack: "build"` if the file lacks it (this is often the first write on a resumed-from-`mission.md` project), `phaseCeremony: "full"` or `"narrow"` (from the card's approval choice).

No user gate — the card was the contract; on return do NOT yield or summarize-and-stop — hand straight back so the orchestrator auto-continues in the same turn — to build-design on full ceremony, straight to build-backend on narrow (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/auto-continue.md`). Downstream skills recompute the hash on entry; on mismatch they surface once (requirements changed since approval → *Continue / restart spec*), never auto-block. For `phaseCeremony: "narrow"`, the orchestrator's Loop control skips build-design/design-compliance entirely and routes straight to build-backend (unaffected — same `requirements.md`+`plan.md` contract, just thinner); build-review runs its `standalone-dogfood` mode instead of pipeline-review, and still writes `phase-complete`/`phase-blocked` on exit (see build-review's Mode detection). Convey to the orchestrator: phase spec written (and drift-reviewed, if full); the spec dir and branch; ceremony scope chosen and why; counts (stories, screens, API contracts, plan groups, validation checks where applicable); how many drift issues were fixed against the card (full only).

---

## Mode: replan (between phases)

Run in the **same session** the phase completed. Reconcile every living doc to **as-built** per its discipline in `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/living-docs.md` — keep each current and within its boundary, never duplicating across docs. No doc-by-doc narration here; the schema owns each doc's update rule. The load-bearing rules:

- **`product.md` is the phase-start drift anchor — keep it as-built.** New screens → add rows, Status `built`. Cut screens → keep the row, Status `removed` + one-line why. Reshaped screens → Status `changed`, note how. Regenerate the App Map + Navigation Structure. Never leave a removed/pivoted screen showing as planned.
- **`tech-stack.md` is the widest-read doc — keep it as-built.** New dependency/library/service → add it with its pinned version. Changed pattern/decision → update the Key Technical Decisions table and cross-link the `docs/decisions.md` entry holding the *why*. Anything ruled out earlier but adopted (or dropped) → correct it. A stale line here misleads every phase skill.
- **`mission.md` is constitution-frozen — do NOT silently edit.** A phase that outgrew the mission is a pivot: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/pivot-protocol.md`.
- **`docs/api.md`, `docs/decisions.md`, `docs/architecture.md`, `README.md`, `CLAUDE.md`** — update per living-docs.md (api.md always-current; decisions.md gets new non-obvious calls; CLAUDE.md index + directive sweep — append durable project-scoped user directives stated this phase, 0–2 lines typical).

**Backlog triage (silent — no user interaction).** In `backlog.md`: completed `open` item → `done YYYY-MM-DD`; `DF-N` verified fixed → `resolved YYYY-MM-DD` (obsolete/non-reproducing → `dropped`); superseded/off-roadmap task → `dropped`; an open item that merits a full phase → note it for the roadmap; leave genuinely-pending items `open`. Write it back before the changelog.

**Changelog — the divergence ledger.** Generate `CHANGELOG.md` from the git log grouped by date (or `scripts/changelog.py` if present); strip merge commits and branch housekeeping; under this phase's date heading add explicit `Pivoted:` / `Removed:` lines for anything that diverged (git's additive log won't show these).

**Merge.** Merge the feature branch to `main` with a no-fast-forward "Phase N complete: [feature]" commit and delete the branch. Fire the phase-wrap brain trigger before the merge (once per phase).

One `AskUserQuestion` batch may cover: did Phase N deliver as planned (scope slipped?); roadmap priority/sequence changes; constitution changes; confirm Phase N+1 is the next roadmap feature. Zero questions is acceptable if nothing changed. The orchestrator owns the next-phase write.

---

## Ground rules

- **The user approves outcomes, never spec files.** constitution's gate (the product story) is the orchestrator's; phase's gate is the Outcome Card (yours). The only other user contact is the immediate **fork** on a `felt-impact` decision — via `AskUserQuestion`, in outcome language.
- **Two research jobs (R1).** Product research once at shape → `research.md`; feature research every phase → here, before drilling. Never assume the former covers the latter.
- **Demand was validated at the shape gate** — do not re-drill it in constitution.
- **External-integration Phase 2+ rule (no exceptions).** Any feature needing third-party OAuth, API keys, webhooks, or external accounts (Slack, SendGrid/SES, Twilio, Stripe, social login) belongs in Phase 2+ — they multiply Phase 1 risk. Test: can the core goal be demonstrated without it? If yes, defer. Push back if the user insists.
- **Scope discipline (no exceptions).** The card contains ONLY what the user explicitly asked for or confirmed. Related ideas surfaced in drilling or research → Phase 2+ Out of Scope, not this phase. "Just the basics" / "keep it simple" = a hard ceiling. Never infer scope from app-type norms.
- **Success must name the pain.** "Success looks like" references the specific problem described. If the user said "WhatsApp chaos," success = "no more WhatsApp for ops notes" + "manager finds last week's notes in 10 seconds" — not "notes are saved and visible."
- **Every schema field filled.** An empty section is a failure — write it or remove the heading. API contracts include all error conditions, not just success. Every user story has a validation check.
- **Agents never commit** and never start servers; the orchestrator owns commits and process lifecycle. Every leaf brief carries the containment line. Verify every leaf's outputs on return (`subagent-policy.md` Rule 5).
- **Settled forks are canon.** Check `docs/decisions.md ## User decisions` before surfacing any fork; record every resolved one; never silently overturn one after compaction.
