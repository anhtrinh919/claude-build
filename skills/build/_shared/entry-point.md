# Entry point — one door, always the orchestrator

Every build project has ONE entry point: the orchestrator, named in `.build-state.json`'s `stack` field (`"build"` → `/build`, `"build-lite"` → `/build-lite`). Sub-skills are steps *inside* it, never doors of their own.

**On any turn where `.build-state.json` exists:**
1. Read it. The `stack` field names the orchestrator that owns this project.
2. If you have not loaded that orchestrator this session, load it FIRST — it owns the resume ladder, the ground rules, the auto-continue contract, and the handoff wiring. Resuming straight into a sub-skill (build-spec, build-design, …) skips all of that; that is the exact bug this rule prevents.
3. The orchestrator reads `step`/`currentSubStep` and routes to the right sub-skill in the right mode. You never pick a sub-skill off the state file yourself.

**Missing `stack`.** A project started before the field existed, or one whose state file was rebuilt from `mission.md`, has none. Don't guess from the step names. **Whoever makes the next state write backfills it** — the orchestrator, or a sub-skill if a sub-skill writes first (build-spec writes `spec-complete` before the orchestrator writes anything on that path). Every state write either backfills `stack` or preserves it; none may drop it.

**Exceptions — genuinely standalone, user-invocable directly:** build-shape `verdict` mode, build-review `standalone-dogfood` mode, build-polish, build-migrate, build-deploy re-deploy mode (an already-shipped project, entered at `deploy-complete`). The first three need no `.build-state.json` at all; build-migrate and re-deploy read one but declare their own contract and are entered directly, not routed. These declare their own standalone contract and need no `.build-state.json`. Everything else is orchestrator-only.
