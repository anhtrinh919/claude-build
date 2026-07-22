# Changelog

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
