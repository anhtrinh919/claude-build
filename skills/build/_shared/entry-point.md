# Entry point — one door, always the orchestrator

Every build project has ONE entry point: the orchestrator, named in `.build-state.json`'s `stack` field (`"build"` → `/build`, `"build-lite"` → `/build-lite`). Sub-skills are steps *inside* it, never doors of their own.

**On any turn where `.build-state.json` exists:**
1. Read it. The `stack` field names the orchestrator that owns this project.
2. If you have not loaded that orchestrator this session, load it FIRST — it owns the resume ladder, the ground rules, the auto-continue contract, and the handoff wiring. Resuming straight into a sub-skill (build-spec, build-design, …) skips all of that; that is the exact bug this rule prevents.
3. The orchestrator reads `step`/`currentSubStep` and routes to the right sub-skill in the right mode. You never pick a sub-skill off the state file yourself.

**Exceptions — genuinely standalone, user-invocable directly with no active build:** build-shape `verdict` mode, build-review `standalone-dogfood` mode, build-polish. These declare their own standalone contract and need no `.build-state.json`. Everything else is orchestrator-only.
