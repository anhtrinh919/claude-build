# Changelog

## 3.0.0 — 2026-08-09

**Breaking: `docs/decisions.md` is retired and replaced by `docs/rejected.md`.** The old file mixed two things with opposite lifetimes — permanent history and current product state — in one append-only doc. State drifts; history does not. So every entry that described *how the product works* started contradicting the living doc that owned the same fact, and the ledger became a second, stale opinion injected at every phase start and after every compaction.

Eight live projects were measured before the change. In the two largest, over 90% of the file was out of scope. claws held 83 headings, of which one was `## User decisions` — the rest were `## Phase N backend — implementation notes` and `## Phase N review — findings and fixes`, a build log that `CHANGELOG.md` already owns. mq-expense's file opened by declaring itself a copy of the tech-stack table, which was exactly what the scaffold step told it to do. mama-health stored CLI flags and a JSON payload shape as decisions — an entry that goes wrong the day a flag is renamed. layout-svg carried an entry marked superseded inline, with the full original reasoning above the reversal, leaving a reader to work out which half was live. Two projects had independently hand-invented the fix: lune-startup grew a `### Superseded, kept so they are not re-argued` section the schema never defined, one line per item, naming only what was dropped.

**The new scope is structural, not a discipline rule: an entry describes a question, a living doc describes the product, and no sentence does both.** Two docs can only conflict when both claim the same fact, so a file whose subject no living doc discusses cannot conflict with one. What survives that test is exactly what nothing else records — which options a settled fork killed, and that the question is closed. Both are permanently true. The chosen option is gone from the ledger; it lives where it is implemented.

Four rules carry it, and each one closes a failure found in the measured files: **one line per entry** (layout-svg ran past 200 words of implementation reasoning), **nothing in the file but entries** (the old schema gave a format for entries but never said the file held only entries, so 1100 lines of diary walked in), **never the chosen option or a mechanism**, and **append-only with no rewriting** — a reopened question gains `· reopened Phase N` on its existing line, because a rejected option stays rejected whatever the product later does.

**Everything evicted has a named home.** Phase narration, implementation notes, and review findings → `CHANGELOG.md`. Stack and dependency calls → `tech-stack.md ## Key Technical Decisions`, which already had Why and Alternatives-Rejected columns and was already the widest-read doc. Component and data-model calls, and the behavioral-mechanism sequence diagram → `docs/architecture.md`. A standing user directive → `CLAUDE.md`. A phase's non-visual design notes → a new `specs/<phase>/design-notes.md`, phase-scoped so it expires with the phase instead of accumulating forever in a global file.

**Two maintenance mechanisms are deleted outright, not replaced.** The replan reconcile pass is gone — there is nothing left in the file that can go stale. So is the supersede-and-rewrite protocol, which the schema had mandated and which no skill ever implemented.

**`/build:migrate` converts existing projects, and now runs on current-schema projects too** — previously they were a no-op. It sorts every old entry to one of the five destinations above, writes the new ledger, then archives the original whole and unedited to `docs/archive/`, where no skill reads it. Nothing is deleted. It is idempotent, and it refuses to invent a rejected option an entry does not state.

## 2.1.0 — 2026-08-08

**The app-shell spec is deleted; the stack no longer prescribes a shell.** `references/app-shell-spec.md` specified one generic SaaS shell in literal values — a 256px sidebar, Cmd+B to collapse, a 56px bottom nav, Sonner toasts bottom-right for 3–5s, a fixed settings category list. It did not help in practice.

The catalogue was only half the problem. `validation.md`'s Phase 0 block restated those same prescriptions as pass/fail checks, so a product that made a different and perfectly reasonable choice was graded as failing its own shell. A prescription plus a checklist that enforces it is worse than either alone.

`requirements.md ## App Shell` now asks the phase to *state* its navigation, auth, settings, and universal-pattern decisions, with no catalogue to copy from. `validation.md` writes one check per decision the spec actually states, and never a check for a pattern the app did not choose. The shell survives as Phase 0 scope in `_shared/roadmap-axis.md`; what is gone is the claim that every product needs the same one.

## 2.0.0 — 2026-08-08

**Breaking: every skill drops the `build-` prefix.** The plugin is already named `build`, so invoking the design skill meant typing `/build:build-design`. Skills are now `/build:spec`, `/build:design`, `/build:backend`, `/build:review`, `/build:shape`, `/build:polish`, `/build:migrate`. The orchestrator keeps its name (`/build`) — its directory also holds `_shared/` and `schemas/`, so renaming it would have rewritten 46 internal paths for no user-facing gain. `/build:migrate` sweeps this rename in existing projects: its detection ladder no longer stops flat on a current-schema project, it greps for a `build-` token and rewrites the references it finds. Without that, every project's own `CLAUDE.md` would keep naming skills that no longer answer.

**The deploy milestone is gone.** `build-deploy` existed to run a whole-app review, a whole-app dogfood, a commit check that was already a no-op (every phase merges at replan), and a Vercel deploy. Only the unfenced dogfood was unique, and it now runs inside the final phase's review: when the orchestrator signals the last roadmap phase, the dogfood walk drops its scope fence and covers every Named Flow and every prior phase's outcomes. That is the stack's only cross-phase integration check. `roadmap-complete` is now a declared terminal state — `1.0.0` shipped it as a state written by nothing, and the inverse failure (written by something, handled by nothing) was the risk here.

**The seven-lens bad-idea gate is gone.** `build-shape` had a cold-shower gate that looped until the user picked Proceed / Refine / Drop, plus a standalone verdict mode. It was ceremony — a gate that existed to be passed. Shaping is now two steps: a concept interview, then 3C research. Nothing in the stack now pushes back on an idea; that is deliberate. The `shape-complete` write moved to the end of the research step, which was the one thing that would otherwise have silently broken the resume ladder.

**`ux-rules.md` deleted — 11364c that no step ever loaded.** It appeared only in a References list and as a cross-link; no skill read it during a run, while the rules it held were written out a second and third time in `gates.md` and `app-shell-spec.md`. Its ungated rules moved to the docs an agent actually opens. Two wrong citations died with it: a gate tag pointing at the wrong rule range, and a rule ID that never existed.

**New `references/design-tokens.md` closes a real gap.** `G-TOKENS` is the *checker*, and it lives in `gates.md`, which only the external design mode loads — so the `claude-code` mode had no token standard at all. The new file loads at both modes' close-out, where `design-tokens.css` is written.

**`drilling-discipline.md` was two versions behind its own source.** The repo held 14056c against a 6449c canonical file, with mismatched header hashes, on a file marked do-not-hand-edit. Re-synced. The current source drops the published-artifact interview and standardizes on batched `AskUserQuestion`; no skill depended on the artifact mechanism.

**`claude-build-lite` is retired** — GitHub repo archived, `stack: "build-lite"` removed as a state value.

**The whole stack was rewritten to ASD-STE100 under measured char budgets: 288138c → 192886c across 31 files, a 33% cut.** The 8 SKILL.md files came down 38%, the 23 supporting docs 24%. Cuts targeted rationale prose, worked examples that restated a rule already given, and step-by-step hand-holding a current model derives from the goal. Every gate, contract, path, command, threshold, and conflict rule was kept; schema templates were diffed field-by-field before and after, because a lost template line is a field an authored document silently drops. Several files stopped above budget at a genuine fidelity floor — `constitution.md` is 87% template, `claude-md.md` 93%, `validation.md` 87% checklist.

**Structural consolidation.** Three constitution schemas merged into `schemas/constitution.md`, closing two orphan schemas that `build-spec` spawned a drafter to author without handing it either path. Two drafter briefs merged into `briefs/drafter.md`. `pivot-protocol.md` deleted — its own text said it invented nothing, and that verified. Canon is now cited, never restated: a 9-gram scan across every file pair returns only citation paths and the standard's own contract header. Skills share one spine and one vocabulary — `Mode:` for a branch, `Step N` for a sequence — enforced by a 7-point conformance check in `docs/skill-standard.md`.

## 1.4.0 — 2026-07-29

**Phase 0 scoped down to shell + hero screens.** It used to mandate the entire planned UI built to polished static before any phase 1 work started — expensive, and it made every reachability gate (`G-REACH`) check against the full end-state inventory instead of what actually exists yet. Phase 0 now builds only the shell + hero screens (home/dashboard + 1-2 top nav screens, Claude's call); every gate that checks orphans/dead-ends now scopes to "screens built so far" — a deferred screen is bookkeeping, not a bug. Vertical slices that touch a screen Phase 0 didn't cover build it to static first, same phase.

**Design-track stickiness.** The claude-code/external design-tool choice used to be re-derived every phase from scratch — a project could flip tools mid-build with no real reason to. If Phase 0 picked `claude-code`, every later phase now reuses it automatically, no re-ask, regardless of how much new visual vocabulary that phase introduces. `external` keeps its existing per-phase recommend-and-ask logic.

**Dogfood gained a structural nav check.** The existing gates covered orphan screens on the external track's static exports only — never the live app, never `claude-code`-track screens post-backend. Phase 1's dogfood pass now runs three checks against the running app before any story walk: way back to home, settings/account reachable if the product has user-configurable state, current location legible without guessing.

**Living-docs reclassified by actual approval status.** Several docs were called "frozen after approval" when the user never saw or approved them — only the Outcome Card is genuinely shown to the user verbatim and approved. `mission.md`/`requirements.md`/`plan.md`/`validation.md`/the design hand-off docs are now correctly framed as agent-authored snapshots, updatable without ceremony as reality diverges (a user-*felt* change still routes through the standard felt-impact-fork rule; `requirements.md`'s within-phase drift-hash contract is untouched — that's a design/backend execution guarantee, not a user-approval claim). Caught and fixed two other files that still asserted the old "frozen" framing (`schemas/living-docs.md`'s own `docs/api.md` description, `schemas/claude-md.md`'s scaffold text) — both were stamping the reversed doctrine into every new project's `CLAUDE.md`.

**Mobbin MCP grounds competitor research and claude-code design hand-offs** in real shipped screens instead of write-ups and training-data priors alone.

**Best-practice tightening:** project directives are captured the instant they're stated, from any skill, not just at replan; validation checks are held to an observable, walkable-sentence standard; outcome-card exclusions are named and concrete, not categorical; `docs/decisions.md` is now a stated input to both review skills, with settled decisions read, not re-litigated; `GLOSSARY.md` added as a conditional living doc, `INTERACTIONS.md` evaluated and deliberately skipped.

**One real regression caught by the fresh-eyes review pass this batch of fixes required:** `build-spec/SKILL.md`'s narrow-ceremony table still justified its "zero new screens" check by "Phase 0 built the full IA" — stale, contradicting the hero-screens-only scope above. Fixed to reflect that a screen Phase 0 didn't build needs design first.

## 1.3.1 — 2026-07-26

Fresh-eyes review of 1.3.0 caught a real regression: phase mode's Scope grill still asked "who uses this feature specifically" — a trunk-level actor question — despite `phase-scope` mode's own gate table saying trunk always collapses there since the actor was already settled at constitution. Removed; the roadmap-entry root is now explicitly marked cited-not-re-derived, and the remaining branch/leaf questions are level-labeled so the drilling actually routes through the gated mechanism instead of reading as an unlabeled checklist next to it.

## 1.3.0 — 2026-07-26

**The drilling mechanism rewritten — root cause of three reported bugs was the same self-contradiction.** The tree's own Parent Test says a child may not exist until its parent is answered, yet the four-step shape demanded the *whole* tree be authored before anyone answered anything — that's what produced stray off-tree questions (a second, untested "trunk/branch/leaf" ritual lived inside `build-shape`'s Step 1.1, explicitly exempted from the validity tests), drift across batches (no committed structure to return to), and irrelevant artifact rows (children guessed for a parent answer that hadn't happened). Fixed: reveal one level at a time — root → trunk → branch → leaf, walked depth-first, one open thread at a time — each level derived only from the answer just given, gated by both the existing structural tests and a new mode-specific semantic gate. A level that fails either gate collapses instead of being manufactured to complete the shape. `decision-tree.md` is now a living document appended to as each level resolves, not authored once upfront; anything discovered mid-session (previously routed through `build-spec`'s undocumented "Latent decisions" side-channel) is now appended into the same tree instead of asked as a bare `AskUserQuestion`.

**Stress-tested against three real grilling contexts before locking in one vocabulary.** A single semantic definition (borrowed from Impact Mapping / Opportunity Solution Trees — outcome/actor/impact/deliverable) fits a brand-new concept but not a scope decision on an already-decided phase or a technical/architecture call — forcing it everywhere would have reintroduced manufactured structure in the two mismatched cases. Three gate presets now exist instead: `concept-shape` (build-shape's Step 1.1), `phase-scope` (build-spec's phase mode and Step 1.4, where trunk always collapses since the actor was already settled upstream), and `technical` (constraint/tradeoff-axis/approach/implementation-choice, ADR-flavored, for infra and architecture calls).

**Single source of truth.** The discipline previously lived as three independent, hand-maintained copies (this file, build-lite's copy, and `/grill-me`'s own restatement) — two were already byte-identical, the third had drifted in wording. Canonical copy now lives at `~/.claude/skills/grill-me/references/drilling-discipline.md`; this file is a generated, header-stamped copy produced by `sync-drilling-discipline.sh` — do not hand-edit it.

## 1.2.1 — 2026-07-26

**Drilling discipline rewritten as a four-step shape.** What was a single list of rules for tree-walking a conversation is now sequenced: batched orientation (`AskUserQuestion` in 4-5 question groups) → author `decision-tree.md` → convert it into a fillable artifact → analyze the returned answers as a set (contradictions, rejected options, revealing blanks) before folding anything into docs. Added explicit validity tests for deriving a tree (parent, fan-out, single-child, settled-fact) so a tree is derived, not imposed. `build-shape`'s Step 1.2 now applies the discipline for question craft only — its gate carries too few forks to justify the full four-step shape (no `decision-tree.md`, no artifact); `build-spec`'s Step 1.4 and phase-mode scope grill apply it in full, phase-mode scaled to the phase's actual decision count.

## 1.2.0 — 2026-07-22

Audit-and-repair pass over 1.1.0: eight full-read auditors across every skill file, then three adversarial review rounds. 1.1.0 shipped several half-landed fixes; these are the rest.

**Narrow-ceremony phases were unbuildable.** The resume ladder's `spec-complete` row routed unconditionally to build-design, so a cold resume ran the design pass a narrow phase exists to skip — then design-compliance graded screens it never redesigned. `build-backend` separately stopped and told the user to run `/build-design` on exactly the phase the orchestrator skips: a hard stall. Both closed; a narrow phase's design source is now stated as the prior phases' output, and a narrow phase needing an undesigned screen surfaces as a scoping error instead of guessing. `standalone-dogfood`, which closes narrow phases, gained the automated-checks + `code-reviewer` rung it lacked — a narrow phase could reach `phase-complete` with a red typecheck and zero code review.

**`roadmap-complete` was written by nothing**, making Milestone 3 unreachable except by invoking `/build-deploy` by hand. The `phase-complete` row now writes it when no roadmap phase is left, runs replan first (so the final phase's branch actually merges), and continues into deploy without a second go/no-go.

**`stack` only landed on one path.** It was written at the first state write and nowhere else, so a project resumed from `mission.md` with no state file stayed permanently stackless — the exact misroute the field was added to prevent. Every state write now backfills it, including build-spec's and build-migrate's; build-migrate also stops carrying the retired `baselines` field into upgraded projects.

**`gates.md` finished its conversion.** 1.1.0 relabelled it the external track's catalogue but left the checks written against live code — grep the CSS, run axe, load the mockup in a browser. Every check is now native to exported static images; what a flat image genuinely cannot answer (focus states, keyboard order, motion) reports `— not observable`, names its build-time check, and does not block, so an external design can't be blocked forever by an unanswerable gate. `G-REFERENCE` no longer tells the agent to fix a design that belongs to the user.

**Contract repairs.** `build-backend` and `build-review` no longer cite `handover.md` on the claude-code track, where it doesn't exist. `build-polish` lost an entry guard that pointed it through an orchestrator with no route back. `build-deploy`'s re-deploy mode is named in the exemption list. The external design hand-off is a named stop in `auto-continue.md`, and a returning user's brief is no longer silently rewritten. `living-docs.md` credits `build-design` rather than a `/frontend` skill retired months ago.

## 1.1.0 — 2026-07-22

Three cleanups to the /build stack.

**claude-code design track = impeccable, whole.** The `claude-code` track is now nothing but "invoke the `impeccable` skill and let it work" + a close-out (write `design-tokens.css` + non-visual decisions). The milestone/gate-report/structure-map/self-critique scaffold, the visual-shown log, and the internal design-brief schema are gone — impeccable owns structure, a11y, craft, the mockups, and the live show-and-fork. The `external` track (brief → wait → review vs `gates.md` → `design-comment.md` → screen→image index) is unchanged; `gates.md` reframed as the external track's catalogue. Deleted `references/toolkit.md` and `schemas/design-brief-internal.md`.

**Cut Safety Defaults, baselines, and phase-type.** Removed SD1–SD5 and the standalone `baseline` mode from `build-spec`, the four `*-baseline.md` reference files, the `baselines` state field, and the `initial`/`feature`/`rebuild` phase-type across `build-backend`/`build-review`/`build-deploy`/schemas. Shell/regression logic is now keyed on phase number (Phase 0 vs 1+). `build-deploy` reads the hosting target from `tech-stack.md ## Choices` and asks at deploy time if unset.

**Flow control — one door, one turn per phase.** New `stack` field in `.build-state.json` names the owning orchestrator so a resume never jumps straight into a sub-skill; every non-standalone sub-skill carries an entry guard routing back through `/build`; a single auto-continue contract (`_shared/{entry-point,auto-continue}.md`) names the only stop points (the user gates) and holds `/eli` to those boundaries.

## 1.0.3 — 2026-07-22

claude-code design track now invokes the `impeccable` plugin for craft judgment (register, color, type, layout, motion, anti-slop) instead of an inline `craft.md` ruleset (deleted). Milestones, gate report, forks, and backend handoff unchanged. README/plugin.json/marketplace.json corrected to list all nine skills (`build-deploy` was missing from the count).

## 1.0.2 — 2026-07-22

Milestone 1 restructured into six literal steps (concept interview, 3C research, bad-idea gate, product interview, constitution writing, roadmap), each resumable on its own. New Milestone 3 (`build-deploy`): whole-codebase review, whole-app blind dogfood, merge verification, real deploy step. New consolidated pivot-protocol doc. Drilling-discipline and roadmap-axis doctrine extracted to `_shared/`.

## 1.0.0 — 2026-07-06

Complete rebuild of the SDD stack, published as the `build` plugin.

- Eight skills (`build`, `build-shape`, `build-spec`, `build-design`, `build-backend`, `build-review`, `build-polish`, `build-migrate`), invoked as `/build*`.
- State-file-driven (`.build-state.json`), re-entrant: resumes mid-phase after context loss.
- Inline orchestration (no background foundation build); leaf subagents only.
- One shared browser-review engine; one UX / craft / accessibility gate catalogue.
- Optional brain-memory integration, guarded to skip cleanly when the memory system is absent.

Prior 0.x releases (a first-generation stack under different skill names) are preserved in git history.
