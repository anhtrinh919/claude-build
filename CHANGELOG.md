# Changelog

## 1.0.0 — 2026-07-06

Complete rebuild of the SDD stack, published as the `build` plugin.

- Eight skills (`build`, `build-shape`, `build-spec`, `build-design`, `build-backend`, `build-review`, `build-polish`, `build-migrate`), invoked as `/build*`.
- State-file-driven (`.build-state.json`), re-entrant: resumes mid-phase after context loss.
- Inline orchestration (no background foundation build); leaf subagents only.
- One shared browser-review engine; one UX / craft / accessibility gate catalogue.
- Optional brain-memory integration, guarded to skip cleanly when the memory system is absent.

Prior 0.x releases (a first-generation stack under different skill names) are preserved in git history.
