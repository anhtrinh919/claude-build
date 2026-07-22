# Changelog

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
