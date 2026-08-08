# Roadmap — [Project Name]

## How phases are sliced (read before filling)

Phase 0 scope (always first, automatic) is defined in `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/roadmap-axis.md` — read it there, not here.

**Phases 1–n — Vertical slices (one feature each).** Each phase wires one feature end-to-end (screen → API → data), tested and polished before the next starts.

**Slice test — every Phase 1+ must pass:** after this phase, can the user *do* something new end-to-end? "No, but a layer is now in place" = invalid slice.

**Banned (horizontal phases):** "build the backend", "all data models", "all APIs" — these deliver a layer, not a feature. Such work belongs in Phase 0 or inside the first slice that needs it.

**Sizing:** one feature or flow per slice — small enough to build, test, and polish in one pass. Two unrelated features → split.

## Phases
Phase 0 is fixed; user (as PM) shapes Phase 1+ order.

0. **Phase 0 — Foundation:** Scaffolding + app shell + hero screens (polished static, mock data). The core shell, clickable and visually final; nothing wired yet.
1. **Phase 1 — [Feature Name]:** [The one feature this slice brings to life, end-to-end. Why it comes first.]
2. **Phase 2 — [Feature Name]:** [The one feature this slice delivers. Why here.]
3. **Phase 3 — [Feature Name]:** [...]

## Global Out of Scope
What this project will explicitly never do. Named precisely — not vague.

- [Specific exclusion]
- [Specific exclusion]
