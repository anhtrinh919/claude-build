---
name: spec
description: 'The SDD spec authority: drills the user, then authors the docs. Mode arg — **constitution** (product interview, constitution docs, roadmap), **phase** (feature research, scope drill, Outcome Card gate, requirements/plan/validation), **replan** (living docs, changelog, merge). Invoked by /build.'
---

# /build:spec — drill, then author

Drill the user for every decision the build needs, then author the docs those decisions imply. You ask; leaf agents draft; you write.

Enter through `/build` (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`). Plain language to the user — never a path, schema name, or stack term (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

The mode bodies are ladders, not scripts. Push deeper when answers are thin; stop when a topic is settled.

## Invocation contract

| Mode | Model | Mechanism | Reads | Writes | Terminal step |
|---|---|---|---|---|---|
| **constitution** | Opus + Opus drafters | inline | Product Shape + `research.md` | `mission.md` `product.md` `tech-stack.md` `roadmap.md` + living-docs scaffold | **none** — `step` rides on `shape-complete`, `currentSubStep` tracks 1.4/1.5/1.6. The orchestrator writes `constitution-complete` after its gate |
| **phase** | Opus + 1 Sonnet research leaf + 2 Opus drafters | inline | constitution docs · `roadmap.md` · `backlog.md` | approved `outcome-card.md` + `specs/YYYY-MM-DD-[slug]/{requirements,plan,validation}.md` | **`spec-complete`**, with `requirementsHash` |
| **replan** | Opus | inline | the just-completed phase · `roadmap.md` | living docs as-built · `CHANGELOG.md` · merged branch | none |

The orchestrator passes the mode explicitly. Act on the arg; never auto-detect.

## The two research jobs

- **Product research** — once, in /build:shape → `research.md`. The whole product at idea level.
- **Feature research** — every phase, here, before drilling. How comparable products handle *this phase's* feature set.

`research.md` never covers per-phase scope. Feature research runs every phase regardless.

## Division of labor

The user decides what to build, for whom, scope, priorities, success — **and any technical call they will feel.** You decide how: framework, file structure, API shape, libraries, tests, naming. Test and fork form: `voice.md`.

The trap: a choice can look like plumbing and still be felt. "Cache nightly vs. read live" is instant-but-stale vs. fresh-but-slower — *that consequence* is the user's. Surface the felt tradeoff, never the technology.

## Drilling discipline

`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/drilling-discipline.md`, at full depth. Two presets:

- **Step 1.4** — `concept-shape`, continuing /build:shape's tree. Root and trunk are settled there; resolve the remaining branches and leaves. Never re-open root or trunk.
- **phase scope grill** — `phase-scope`. Root is the phase's outcome, cited not re-derived. Trunk collapses to branch, because the actor is settled upstream.

Run a level only when the open decisions warrant it. `decision-tree.md` feeds the Outcome Card — a working document, never a deliverable.

## Latent decisions

Every drafter returns a `## Latent decisions` list of choices it made that the drilling did not reach; two to six is healthy, zero means it did not look. Route *their* calls, do not re-derive:

- **felt-impact** (the user sees it, feels it, waits on it, or is constrained by it) → append as a leaf under its real parent in `decision-tree.md`, ask it through that tree batched with related items, fold the answer in before writing, log the rejected options to `docs/rejected.md`.
- **invisible-plumbing** → never ask. Record in the doc that owns it — `tech-stack.md ## Key Technical Decisions` or `docs/architecture.md`. Never `docs/rejected.md`.

Unsure → treat as felt-impact. Check `docs/rejected.md` before surfacing any fork; a question listed there is closed.

## Brain integration

`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md`, `$AGENT=spec`, `$TAGS` from `tech-stack.md` (constitution has no stack — pass empty). **Friction trigger:** the user re-answers or rejects the same topic 3+ times, or edits a spec after approval. **Phase-wrap trigger:** once, at the end of replan, before the merge.

---

## Mode: constitution

/build:shape passes a locked **Product Shape** and `research.md`. Those decisions are settled — confirm each in passing, drill the detail within it. A drill answer that genuinely contradicts the shape is a shape change: surface it and confirm, never silently re-derive.

### Step 1.4 — Product interview

Write `currentSubStep: "spec.1.4"`. `step` stays `shape-complete` all through this mode.

Two topics only, at full drilling depth: **mission** (what this does for the user in one outcome sentence, what it explicitly does NOT do, the experience it creates) and **tech constraints** (genuine business constraints, existing infrastructure, technologies explicitly ruled out — decide everything else silently). Auto-move to 1.5.

### Step 1.5 — Constitution writing

Write `currentSubStep: "spec.1.5"`.

**Master User Journey** — three layers, becoming `## Master User Journey` in `mission.md`. Every phase's user stories cite it, and the final phase's unfenced dogfood walks it.

- **Core Jobs:** per actor, 1–2 statements, "When [situation], I need to [goal], so I can [outcome]." Three to five total, motivation-level, no feature names.
- **Named Flows:** 3–6 flows, each 3–5 steps at verb-noun granularity (**Onboarding:** Register → Connect data → Configure → First run). Draft from the conversation, then one `AskUserQuestion` (complete / missing something / needs reordering). Adjust once.
- **Flow-to-phase mapping:** label each step with its delivering phase. A gap usually means a missing phase — note it for 1.6, do not block.

**Author.** Spawn Opus drafter leaves, context-isolated: the Product Shape, `research.md`, the drilling decisions, and the one schema path `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/constitution.md`. They author `mission.md` and `tech-stack.md` in full, plus `product.md` **except** its Screen Inventory phase-label column and App Map phase-coloring — both `TBD — patched at Step 1.6`, since phase numbers do not exist yet. Not `roadmap.md`. They return raw markdown; you write to disk.

Every field filled. `product.md`'s App Map is one Mermaid flowchart regenerated from Screen Inventory plus Navigation Structure, never hand-maintained; its Phase 0 Foundation Scope names hero screens only — home/dashboard plus 1-2 top nav screens, your call.

**Scaffold living docs** per `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/living-docs.md`, headers only — an empty section is fine, fake content is not: `CLAUDE.md`, `backlog.md`, `README.md`, `CHANGELOG.md`, `docs/architecture.md`, `docs/api.md`, `docs/rejected.md`. Every doc under `docs/` opens with the exact line `> Agent context — not for human reading.`; `README.md` and `CLAUDE.md` do not. Seed `docs/rejected.md` with one line per fork already resolved, carrying only what each one rejected. **Never seed it from the tech-stack table** — those choices live in `tech-stack.md` and copying them creates a second, drifting home.

### Step 1.6 — Roadmap

Write `currentSubStep: "spec.1.6"`. **The user drives the slicing — the most important step to get right.** Schema: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/roadmap.md`. Axis doctrine: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/roadmap-axis.md`.

Ask for the feature set roughly — list, do not scope. Draft Phase 0 plus one feature per phase by value and dependency, surface it via `AskUserQuestion` (confirm / reorder / resize / change what comes first), loop until the user is happy, then ask what is globally out of scope and resolve any Layer 3 gap.

**Author `roadmap.md`** — Phase 0 first, then one vertical slice per phase, the user's sequence verbatim, each naming its feature, what it delivers, and why it sits there. **Patch `product.md`** — fill the phase-label column and regenerate the App Map's phase-coloring; a mechanical edit, not a re-author.

**State checkpoint** — `phase: 1`, `feature` (kebab slug of Phase 0), `reviewIteration: 0`, `requirementsHash: ""`, `currentSubStep: null`, `dogfoodPid: null`. `step` stays `shape-complete`. Without this, a compaction during the user's pause restarts at 1.4.

### Return

Return the **latent-decisions list** (felt-impact ones carrying their fork) and a **product story**: what the product does and for whom, what each phase delivers in one line, what it will never do. No paths, no tech terms, no raw Mermaid — mention that `product.md` holds an App Map the user can open in any markdown viewer. The orchestrator owns the gate. **Ask the user nothing at the end of this mode.**

---

## Mode: phase

Read `mission.md`, `tech-stack.md`, and `roadmap.md` first. Never ask a question those docs or the codebase already answer.

### Step 0 — Feature research

Spawn **1 Sonnet leaf** (`run_in_background: false`), context-isolated: the feature name, the product one-liner, and the goal — find 2–3 apps solving *this specific feature*; extract what screens exist, which patterns are standard, what is distinctive.

**Wait for it before asking the user anything** — the grill must be grounded in what it found, not reconciled against it after. On return present the comparison and ask once: use some of these patterns / avoid all / note but design differently.

**Backlog triage.** Read `backlog.md` if present. A `T-N` that fits without expanding the card → mention by ID during orientation. An open `DF-N` → name by ID, ask whether to fix here or separately. Never spend an `AskUserQuestion` on backlog items.

### Step 1 — Scope grill

`phase-scope`. Do not ask a fresh "who is this for" — the actor is settled upstream.

- **Root, cited:** quote the roadmap entry verbatim, confirm it still matches, establish what the user can do at the end that they could not before. An entry that feels wrong is a constitution change — surface it and ask whether to update `roadmap.md` first.
- **Branch:** what is explicitly out of scope though related; what "this works" looks like from the settled actor's perspective.

**Fork-first.** Every decision the user will *feel* is surfaced as a fork **while drilling**, never baked into the card to rubber-stamp at the end — live-filter vs. apply-button, show-all vs. paginate, inline-edit vs. modal. The card records decisions the user already made.

### Step 2 — Stories, screens, primary flow

- **User stories** — "As [actor], I can [specific action] so that [specific outcome]." "Filter the list by supplier so I see only their items" is a story; "manage the list" is not.
- **Screen inventory** — per story, as Screen / State / Key UI elements / Primary user action. Cover empty, loading, error, and mobile states, and the edge cases (double-submit, expired session, back button mid-action).
- **Business rules** — rules governing this feature, patterns from the codebase, tone constraints.
- **Primary flow** — the 1–3 stories whose failure makes the feature useless. These become /build:review's stop criteria; everything else is secondary, where a bug blocks but does not stop the review.

### Step 3 — Ceremony scope

Evaluate before presenting the card. A narrow phase is built directly and dogfooded, skipping spec-authoring and design entirely. Recommend **narrow** only when every check holds; any one false → **full**. Tie-break toward full.

| Check | Narrow requires |
|---|---|
| New screens | zero — every screen this phase touches already exists to polished static. Phase 0 builds hero screens only, so anything it skipped needs design first |
| Primary flow | exactly 1 story |
| Mechanism | no async or background jobs, real-time sync, multi-actor flow, state machine, or role/permission model |
| API surface | ≤3 endpoints, all simple CRUD |

State the verdict in one line of *why* when you present the card.

### Step 4 — Outcome Card gate

**The only user approval at spec time.** Spec files are machine-validated downstream and never shown to the user; the card is their contract.

1. Draft from the drilling session using `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/outcome-card.md`. User language only — no endpoints, schemas, or component names. Leave `approved` unset.
2. Write it to `specs/YYYY-MM-DD-[feature-slug]/outcome-card.md`; create the directory now, date from `date +%Y-%m-%d`.
3. Surface it verbatim, then one `AskUserQuestion` conveying that this card is the contract for Phase N and everything built and reviewed traces back to it. Fork: **Approve — build directly** / **Approve — full spec + design** (recommend whichever Ceremony scope chose) / **Adjust** (fold in, re-surface, re-ask until Approve).
4. On Approve, set `approved: <today>` and `ceremony: full|narrow`. A later card change restarts this mode.

### Step 5 — Scope challenge

List what already exists in the codebase against what is genuinely new, and identify the minimum set of changes. **More than 8 files or 2+ new abstractions → ask whether to reduce scope first.** Note any deferred `roadmap.md` item that bundles in without expanding scope. Author-only.

### Step 6 — Author

Create the branch `phase-N-[feature-slug]` either way.

**Full.** Spawn **two Opus drafters in parallel** from `${CLAUDE_PLUGIN_ROOT}/skills/spec/briefs/drafter.md` — one leaf gets the shared block plus Task A (`requirements.md` + `validation.md`), the other the shared block plus Task B (`plan.md`). Never hand one leaf both. Both work from the drilling decisions verbatim, the approved card, the scope summary, and `tech-stack.md`. Then Steps 7 and 8.

**Narrow.** No leaves, no reconciliation, no drift review. Write `requirements.md` and `plan.md` yourself to the same schemas, every field filled — narrow means less process around the docs, not thinner docs. No `validation.md`. Skip to Step 9.

### Step 7 — Reconciliation (full only)

Check all eight against both outputs; resolve every gap by editing the docs.

1. Every user story maps to ≥1 plan group.
2. Every plan group has ≥1 validation check.
3. Every endpoint appears in plan.md and as an automated curl test in validation.md.
4. requirements.md's data model matches the schema implied in plan.md.
5. Every card primary outcome has ≥1 story and one Outcome check.
6. plan.md implements only what requirements.md scopes; requirements.md scopes only what serves the card.
7. **Endpoint ↔ screen binding.** Every endpoint names a consuming screen or is marked `internal`; every data-needing screen has a backing endpoint. An endpoint no screen calls is scope creep; a data-needing screen with no endpoint is a missing contract.
8. **Screen reachability.** Trace `product.md`'s Navigation Structure across this phase's screens: each must be reachable from home and able to return. Inbound with no way back is a dead end — add the transition.

Then route the drafters' latent-decision lists: merge, dedupe, surface felt-impact as forks now, record the rest. Escalate only a structural conflict needing a product decision.

**Behavioral-mechanism diagram (most phases skip).** A mechanism whose behavior *over time* is not inferable from screens plus endpoints — async jobs, real-time sync, multi-actor flow, state machine, permission model — gets one Mermaid `sequenceDiagram` in `docs/architecture.md`.

### Step 8 — Drift review (full only)

1. **Drift from the card.** Anything serving no card outcome → cut. Any outcome under-served → add. Any contradiction with the constitution or codebase → reconcile. Re-judge the invisible-plumbing calls; promote anything felt to a fork.
2. **Completeness and error paths.** Missing states, unhandled errors, vague API errors, and the "what happens when" holes. Verify the implicit-state list in `briefs/drafter.md` Task A was honored.
3. **Testability.** Stories too vague to verify, checks that cannot fail or do not prove the story, automated checks that are not runnable commands, outcome checks a non-technical person could not verify on screen.

Fix by editing. Surface only a hole you cannot close alone — one `AskUserQuestion` in outcome language, never a raw findings list.

### Step 9 — Write and auto-proceed

Write under `specs/YYYY-MM-DD-[slug]/`: full writes `requirements.md`, `plan.md`, `validation.md`; narrow writes the first two. Then:

1. `sha256sum specs/YYYY-MM-DD-[slug]/requirements.md | cut -d' ' -f1` — taken **after** drift review, never mid-review. Only phase mode writes `requirementsHash`.
2. Write `.build-state.json`: the hash, `step: "spec-complete"`, `phase` + `feature`, `reviewIteration: 0`, `currentSubStep: null`, `dogfoodPid` preserved, `phaseCeremony` from the card, and `stack: "build"` if absent.

No user gate; the card was the contract. Hand straight back (`_shared/auto-continue.md`) — /build:design on full, /build:backend on narrow. Downstream skills recompute the hash and on mismatch surface once (requirements changed since approval → Continue / restart spec) rather than auto-blocking. Report the spec dir, branch, ceremony scope and why, counts, and drift issues fixed.

---

## Mode: replan

Run in the **same session** the phase completed. Reconcile every living doc to as-built per `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/living-docs.md`. The load-bearing rules:

- **`product.md` is the phase-start drift anchor.** New screens → rows, Status `built`. Cut screens → keep the row, Status `removed` plus a one-line why. Reshaped → `changed`, note how. Regenerate the App Map and Navigation Structure. Never leave a removed screen showing as planned.
- **`tech-stack.md` is the widest-read doc.** New dependency → add with its pinned version. Changed decision → update the Key Technical Decisions table in place; it holds the why and the alternatives rejected itself, with no second copy anywhere. A stale line here misleads every phase skill.
- **`mission.md`** — update like `product.md`. Only a user-felt change is a pivot: surface it as a fork, then re-run just the affected constitution slice, never all of Milestone 1.
- **`docs/api.md`, `docs/architecture.md`, `README.md`, `CLAUDE.md`** — per living-docs.md. Append any durable project-scoped directive stated this phase to `CLAUDE.md`. **`docs/rejected.md` is not reconciled** — it is history and cannot go stale. Append only the forks this phase resolved, one line each. This phase's narration — what was built, what was fixed, what pivoted — goes to `CHANGELOG.md`, never there.

**Backlog triage — silent.** Completed → `done YYYY-MM-DD`. `DF-N` verified fixed → `resolved YYYY-MM-DD`; obsolete → `dropped`. Superseded → `dropped`. An item meriting a full phase → note it for the roadmap. Write back before the changelog.

**Changelog — the divergence ledger.** Generate `CHANGELOG.md` from the git log by date (or `scripts/changelog.py` if present); strip merge commits and branch housekeeping. Under this phase's heading add explicit `Pivoted:` and `Removed:` lines — git's additive log will not show these.

**Merge.** Merge the branch to `main` with a no-fast-forward "Phase N complete: [feature]" commit, delete the branch. Fire the phase-wrap brain trigger first.

One `AskUserQuestion` batch may cover: did Phase N deliver as planned, roadmap changes, constitution changes, confirmation that Phase N+1 is next. Zero questions is fine if nothing changed.

---

## Ground rules

1. **Users approve outcomes, never spec files.** Constitution's gate is the orchestrator's; the Outcome Card is yours. The only other user contact is a felt-impact fork.
2. **Two research jobs.** Product research once at shape; feature research every phase, here.
3. **External integrations are Phase 2+, no exceptions** — third-party OAuth, API keys, webhooks, external accounts. Demonstrable without it? Defer. Push back if the user insists.
4. **Scope discipline, no exceptions.** The card holds only what the user asked for or confirmed; everything surfaced in drilling or research goes to Out of Scope. "Just the basics" is a hard ceiling. Never infer scope from app-type norms.
5. **Success must name the pain.** "WhatsApp chaos" makes success "no more WhatsApp for ops notes", not "notes are saved and visible."
6. **Every schema field filled** — write it or drop the heading. API contracts include every error condition; every story has a validation check.
7. **Agents never commit and never start servers.** Every leaf brief carries the containment string from `subagent-policy.md`. Verify every leaf's output on return.
8. **Settled forks are canon.** Check `docs/rejected.md` before surfacing any fork; never re-offer an option listed there. Log every fork you resolve, one line.
