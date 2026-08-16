# Entry point — one door, always the orchestrator

Every build project has ONE entry point: the orchestrator, named in `.build-state.json`'s `stack` field — `"build"` → `/build`. Sub-skills are steps *inside* it, never doors of their own.

**On any turn where `.build-state.json` exists:**
1. Read it. The `stack` field names the orchestrator that owns this project.
2. If you have not loaded that orchestrator this session, load it FIRST — it owns the resume ladder, the ground rules, the auto-continue contract, and the handoff wiring. Resuming straight into a sub-skill skips all of that.
3. The orchestrator reads `step`/`currentSubStep` and routes to the right sub-skill in the right mode. You never pick a sub-skill off the state file yourself.

**Missing `stack`.** A project started before the field existed, or one rebuilt from `mission.md`, has none. Do not guess from the step names. **Whoever makes the next state write backfills it** — the orchestrator, or a sub-skill if a sub-skill writes first (/build:spec writes `spec-complete` before the orchestrator writes anything on that path). Every state write either backfills `stack` or preserves it; none may drop it. **Verify, don't just intend it:** after any write to `.build-state.json`, read the file back and confirm `stack` is present. Absent → the write just dropped it — fix it before doing anything else; never proceed on a state file with no owner named.

**Exceptions — genuinely standalone, user-invocable directly:** /build:review `standalone-dogfood` mode, /build:polish, /build:migrate. The first two need no `.build-state.json` at all. /build:migrate reads one but declares its own contract and is entered directly, not routed. Everything else is orchestrator-only.
