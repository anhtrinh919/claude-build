---
name: build
description: SDD build orchestrator (v2) — re-entrant, state-file-first pipeline that turns an idea into a shipped, dogfooded product one vertical slice at a time. State-first: reads `.build-state.json` to resume mid-phase after context loss; falls back to `mission.md` presence (absent = new project → shaping gate; present = next feature). New project → build-shape (shape + pressure-test + product research) → build-spec constitution (drill + author the four core docs) → user approves the product story → Phase 0 foundation. Each phase (Phase 0 = full polished-static UI + app shell; Phases 1+ = one vertical slice, wired end-to-end): build-spec phase (feature research → drill → user-approved Outcome Card → requirements/plan/validation) → build-design (pick track → design → gate) → design-compliance → build-backend (wave-dispatched, tested) → backend-compliance → build-review (code review + dogfood, silent auto-fix) → user approves the dogfood. Within-phase handoffs auto-continue in the same turn; phase boundaries (constitution, phase-complete) ask a go/no-go. User gates are outcome-only; every felt product decision is a fork the user picks. Trigger on /build, or resume when `.build-state.json` is present.
---

# /build — SDD orchestrator

**Wiring only.** This file holds the state schema, the resume ladder, the transition gates, and the handoff contracts. Every sub-skill owns its own body. Read the tables here to resume — you do not need to open a sub-skill file to know where you are.

Where a quoted string is user-facing, it's the *intent* to convey in plain language, not a script to recite — never file paths, stack names, or jargon to the user (`_shared/voice.md`).

## What this is
- Re-entrant, state-file-first. Every gate writes `.build-state.json`. Any turn can be a cold resume.
- **Sequential** shape → then, per phase: spec → design → backend → review. No background/parallel builds.
- The orchestrator **is the session**: it runs each sub-skill **inline**, in order. The only subagents anywhere are the leaf workers a sub-skill fans out (research, drafters, render, browse, implementation, fix). Never spawn a sub-skill as a subagent.

## Read-first on cold start (once per session, first action only)
Read `.build-state.json` → find `step` in the **Resume ladder** → resume there. Also read `product.md` (screen inventory + App Map = strongest as-built anchor) and `docs/decisions.md ## User decisions` (settled choices = anti-overturn anchor). Skip either if missing (pre-constitution). If the task is clearly not build orchestration, ignore the state file — it's a resume anchor, not a coercion.

## State schema — `.build-state.json`
One owner per field; separate fields prevent write races.

| Field | Values | Owner |
|---|---|---|
| `phase` | number (0 = Foundation) | orchestrator |
| `feature` | slug | orchestrator |
| `step` | see Resume ladder | see *Terminal-step ownership* below |
| `reviewIteration` | fix-loop counter, cap 3 | build-review |
| `requirementsHash` | sha256 of requirements.md (drift detector) | build-spec (phase mode) |
| `currentSubStep` | breadcrumb e.g. `"design.phase-1"`, `"shape.1.2"`, `"deploy.3.4"`; null on clean exit | whichever step is running |
| `dogfoodPid` | running dogfood server PID; null on stop | build-review |
| `baselines` | array of active baseline ids | build-spec (constitution mode) |
| `phaseCeremony` | `"full"` \| `"narrow"` — set per phase at Outcome Card approval | build-spec (phase mode) |

No `foundationStatus` — there is no background foundation build in v2.

**Milestone 1 breadcrumbs.** `currentSubStep` takes values `"shape.1.1"`/`"shape.1.2"`/`"shape.1.3"` during build-shape and `"spec.1.4"`/`"spec.1.5"`/`"spec.1.6"` during build-spec constitution mode — one per literal step (see Resume ladder). A resume mid-step continues the interview at that step, it never restarts the whole grill. **Milestone 3 breadcrumbs** take `"deploy.3.1"`…`"deploy.3.4"` during build-deploy, same convention.

**Terminal-step ownership.** Steps with no gate before the next stage are written by the sub-skill on clean exit: `shape-complete` (build-shape, on Proceed), `spec-complete` (build-spec phase), `phase-complete`/`phase-blocked` (build-review), `deploy-complete`/`deploy-blocked` (build-deploy). Gated transitions are written by the **orchestrator** after its gate: `constitution-complete` (after the user boundary gate), `design-complete` (after design-compliance), `backend-complete` (after backend-compliance) — and any rollback. So build-design, build-backend, and build-spec constitution mode all return without writing a terminal step; `step` simply rides on whichever terminal step preceded them (`spec-complete` through build-design/build-backend; `shape-complete` through build-spec constitution mode) while `currentSubStep` tracks live position, cleared to null on that sub-skill's clean return — the orchestrator writes the next terminal step once its gate passes. `shaping-in-progress` is the one true exception: build-shape has no prior terminal step to ride on (it's the project's first write), so it's a dedicated bootstrap marker, written once before Step 1.1's first question and left in place until `shape-complete` supersedes it.

**Legacy state files.** A `.build-state.json` from the retired v1 stack carries a `foundationStatus` field and/or old enums (`*-approved`, bare `complete`, `frontend-complete`); a project from before the build-2→build rename carries a `.build2-state.json` instead. For a clean resume you may tolerate the old enums (`*-approved` / bare `complete` → the matching `*-complete` / `roadmap-complete`) and rewrite on the next gate write — but if you see `foundationStatus`, a `frontend-*` step, or a stray `.build2-state.json`, run `/build-migrate` first to upgrade the project properly.

## Resume ladder
Within-phase gates **auto-continue in the same turn** — no stop, no phase-wrap ("Phase N is built"), no "paused" framing. Phase-boundary gates present a go/no-go `AskUserQuestion` (Proceed / Stop-for-now) — never tell the user to retype `/build`.

| `step` | Resume at | Gate type |
|---|---|---|
| *(no state, no mission.md)* | build-shape Step 1.1 — write `shaping-in-progress` + `currentSubStep: "shape.1.1"` before the first question | conversation-only until this first write |
| `shaping-in-progress` | build-shape, at `currentSubStep` — continue the interview there, never restart the tree | conversation-continues, no re-gate |
| `shape-complete` | build-spec constitution mode — start at Step 1.4, or resume at `currentSubStep` if one is set (mid 1.4–1.6) | auto (start) / conversation-continues (resume) |
| *(no state, mission.md present)* | build-spec **replan** → feature cycle | — |
| `constitution-complete` | Feature cycle → Phase 0 spec | **Boundary** — `/eli` wrap + AUQ go/no-go |
| `spec-complete` | build-design | Within-phase — auto |
| `design-complete` | build-backend | Within-phase — auto |
| `backend-complete` | build-review | Within-phase — auto (silent handoff) |
| `phase-complete` | check `dogfoodPid` → dogfood handoff if needed → `/eli` wrap + AUQ go/no-go → (Proceed) build-spec replan → next phase spec | **Boundary** — AUQ go/no-go |
| `phase-blocked` | surface open issues — never auto-resume | Always stops |
| `roadmap-complete` | build-deploy — Milestone 3 (whole-codebase review, whole-app dogfood, merge check, deploy) | **Boundary** — `/eli` wrap + AUQ go/no-go into Milestone 3 |
| `deploy-complete` | "Nothing left to build or deploy." Stop. | — |
| `deploy-blocked` | surface open issues — never auto-resume | Always stops |

## Transition gates (automated — replace the old restated auto-continue prose)
After a sub-skill returns, before writing its terminal `step`, run its gate. Pass → write `step`, auto-continue. Fail → orchestrator writes `step` back (rollback) and re-runs the named skill; loop until pass. These never stop for the user.

| Gate | Pass condition | On fail |
|---|---|---|
| **design-compliance** | every screen + every state in requirements.md has a track-aware design artifact — `claude-code`: a mockup in `specs/<phase>/mockups/` (no handover doc); `external`: an exported image in the `handover.md` screen→image index | rollback `step`→`spec-complete`, re-run build-design |
| **backend-compliance** | integration test green for every API contract in requirements.md | rollback `step`→`design-complete`, re-run build-backend |

## Model + mechanism
Every sub-skill runs **inline** (it orchestrates its own leaf workers, or holds a live user gate — either forces inline; `_shared/subagent-policy.md` Rule 1). The orchestrator never spawns a sub-skill.

**Every sub-skill below is a separate, flat, top-level folder in the plugin's `skills/`** — e.g. `${CLAUDE_PLUGIN_ROOT}/skills/build-spec/`, not nested under `build/`. Invoke each by name through the Skill tool (`skill: "build-spec"`, etc.). Only `schemas/` and `_shared/` referenced elsewhere in this file are actually nested under `build/` itself — never guess a sub-skill's path by pattern-matching those.

| Sub-skill | Spawns (leaf workers, subagent-free) |
|---|---|
| build-shape | 1 product-research agent (Sonnet) |
| build-spec | Opus drafters; 1 feature-research agent (phase mode) |
| build-design | render HTML→PNG (Sonnet); optional critic |
| build-backend | implementation agents — Opus (arch/data/API) + Sonnet (leaf/UI/config), wave-dispatched |
| build-review | Sonnet browse; Opus architecture + correctness review; wave-dispatched fix agents |
| build-polish | Sonnet/Haiku dogfood + fix agents |
| build-deploy | 1 `ponytail-review` skill invocation (3.1); 1 `dogfood` agent, unscoped (3.2); Sonnet fix leaves |

**Core docs are Opus-authored** — constitution (mission/product/tech-stack/roadmap), phase specs (requirements/plan/validation), the design brief (design-brief.md). Sonnet/Haiku only *execute a settled plan* (browse, render, implementation, report write-ups). Test: authoring/deciding → Opus; executing → Sonnet. **Ponytail full** governs all implementation work the whole build; it does not apply to core-doc authoring (judged on completeness, not brevity).

## Handoff contracts
Each sub-skill declares its own `## Invocation contract` (model · mechanism · inputs · outputs · terminal `step`) — that is canonical; this table is the quick map. Doc schemas: `schemas/`.

| Sub-skill | Reads | Writes |
|---|---|---|
| build-shape | the idea | **Product Shape** (prose, no file) + `research.md` |
| build-spec · constitution | Product Shape, `research.md` | `mission.md` `product.md` `tech-stack.md` `roadmap.md` + living-docs scaffold + `baselines` |
| build-spec · phase | constitution, `roadmap.md`, `backlog.md` | user-approved `outcome-card.md` + `specs/YYYY-MM-DD-[feature]/{requirements,plan,validation}.md` + `requirementsHash` |
| build-spec · replan | phase-just-completed | living-docs updated, changelog, branch merged |
| build-design | `requirements.md` | `design-brief.md` + `design-tokens.css` + (`claude-code`: `mockups/` + gate report, decisions→`docs/decisions.md`, **no handover**) / (`external`: exported images + `design-comment.md` + `handover.md` screen→image index) |
| build-backend | `requirements.md` `plan.md` `design-tokens.css` + design source (`claude-code`: `mockups/` + `docs/decisions.md`; `external`: `handover.md` + images) | working, integration-tested API |
| build-review | `validation.md` `outcome-card.md`, running app | review report + silent fixes + **dogfood handoff** + `phase-complete`\|`phase-blocked` |
| build-deploy | `roadmap.md` (complete), `tech-stack.md ## Safety Defaults`, `mission.md ## Master User Journey`, every `outcome-card.md` | optional cleanup/fix commits, `docs/deployment.md` or a live deploy, `deploy-complete`\|`deploy-blocked` |

**Narrow-phase note (`phaseCeremony: "narrow"`):** build-spec phase mode's narrow output omits `validation.md`. build-design/build-backend rows above don't apply — design is skipped, backend runs unchanged. build-review is explicitly invoked in `standalone-dogfood` mode instead of the pipeline-review row above, and — only because the orchestrator names this as a phase closure — still writes `phase-complete`/`phase-blocked` on exit (build-review's own Mode detection covers the rest).

**Contracts that never move:** `outcome-card.md` = the user's contract (frozen on approval; a card change restarts build-spec phase mode). `requirements.md` = the machine contract shared by design + backend (hashed). `mission.md` frozen after constitution. `tech-stack.md` is the widest-read doc (carries `## Safety Defaults` + `## Baselines`).

## Loop control
- **New project** (no mission.md): build-shape (Steps 1.1 concept interview → 1.2 3C research → 1.3 bad-idea gate) → build-spec constitution (Steps 1.4 product interview → 1.5 constitution writing → 1.6 roadmap) → constitution boundary gate → Feature cycle at Phase 0. Six literal steps, each its own `currentSubStep` breadcrumb — see Resume ladder.
- **Feature cycle** (per phase): build-spec phase → build-design → *design-compliance* → build-backend → *backend-compliance* → build-review → phase-complete boundary gate.
- **Feature cycle, narrow variant** (`phaseCeremony: "narrow"`, set by build-spec at Outcome Card approval): build-spec phase (narrow — no drafters/reconciliation/drift-review/baseline ceremony, no `validation.md`) → build-backend directly (same `requirements.md`+`plan.md` contract, unaffected) → *backend-compliance* → build-review, explicitly invoked in `standalone-dogfood` mode as this phase's closure → phase-complete boundary gate. build-design and *design-compliance* are skipped entirely — Phase 0 already built every screen to polished static, and a narrow phase by definition touches none that don't already exist.
- **Next feature** (mission.md exists, no active phase): build-spec replan → Feature cycle for the next roadmap phase.
- **Milestone 3 — Deploy** (`roadmap-complete`, no next phase left): build-deploy (Steps 3.1 whole-codebase review → 3.2 whole-app blind dogfood → 3.3 merge verification → 3.4 deploy) → `deploy-complete`/`deploy-blocked` terminal. See `build-deploy/SKILL.md`.
- **Roadmap discipline:** Phase 0 = Foundation (scaffold + app shell + the full planned UI as polished static — every screen, mock data, design-locked, unwired; type `initial`). Phases 1+ = vertical slices (one user-facing capability each, wired end-to-end, built→tested→reviewed before the next; type `feature`/`rebuild`). **Slice test:** after this phase can the user *do* something new end-to-end? Horizontal phases ("build all the APIs") are banned. Never thin a slice — split an oversized one into narrower slices. User is the PM; build-spec drafts the sequence, user confirms.
- **Axis before order.** `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/roadmap-axis.md` — governs Step 1.6's roadmap draft (build-spec constitution mode).

## Ground rules (canon in `_shared/`; one line each)
1. **/eli after every gate write and sub-skill return**, before any further action — auto-continue gates included.
2. **User gates are outcome-only; every felt decision is a fork.** User approves: the shape, the product story (never spec files or `tech-stack.md`), the Outcome Card, the design, the phase-end dogfood. Any decision with felt product impact (UX or performance) → surface as a fork (options + each option's plain tradeoff + a recommended default), front-loaded at the shape/scope/design gate; if it emerges mid-build, ask then — a felt fork overrides auto-continue. Invisible plumbing → decide silently → `docs/decisions.md`. (`_shared/voice.md`)
3. **Settled decisions are canon.** `docs/decisions.md ## User decisions` is the anti-overturn ledger. Check it before re-asking any fork; record every resolved fork. Never silently overturn one after compaction.
4. **Backlog capture.** User defers a request → append a dated one-liner to `backlog.md` immediately and confirm by ID ("Noted as T-7"); dogfood bugs thread as `DF-N`. Refer by ID — deferred asks vanish at compaction.
5. **Agent containment (`subagent-policy.md`).** Every leaf brief carries: no spawning agents, no `/build*`, no `claude -p`, no commit, no server, don't address the user — return content/paths only. Never brief "read and execute a SKILL.md" (that risks a second-level spawn); give a concrete task + file list. After every return: `git log <pre>..HEAD` (unexpected commit = runaway) + `ps ax | grep "[c]laude -p"` (stray worker); kill/reset before continuing. One builder per branch; **agents never commit** (orchestrator commits).
6. **Fix commits state root cause.** One-line "where it's first wrong" + a tag: `[root-cause]` / `[symptom-patch]` / `[heuristic]`. The last two need explicit user opt-in — no silent band-aids.
7. All user questions via `AskUserQuestion`. Cap-hit / blocker surfaces are binary: "Accept anyway, or Stop?"
8. **Pivoting.** `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/pivot-protocol.md` — read when a phase reverses a settled decision or the mission itself. Not a new mechanism: mid-phase is rule 2's fork, post-ship is replan's changelog/decisions convention, mission-outgrown is a constitution-change tripwire.
