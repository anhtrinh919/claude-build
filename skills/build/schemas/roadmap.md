# Roadmap — [Project Name]

## How phases are sliced (read before filling)

**Phase 0 — Foundation (always first, automatic).** Repo + data-layer scaffold + app shell + **the full planned UI built to final visual polish** (every screen, static, mock data, pixel-complete, not wired). Product *looks* finished and is clickable; it just doesn't *do* anything yet. Type: `initial`.

**Phases 1–n — Vertical slices (one feature each).** Each phase wires one feature end-to-end (screen → API → data), tested and polished before the next starts. The visuals exist from Phase 0; a slice brings them to life. Type: `feature` (or `rebuild` for a visual redesign).

**Slice test — every Phase 1+ must pass:** after this phase, can the user *do* something new end-to-end? "No, but a layer is now in place" = invalid slice.

**Banned (horizontal phases):** "build the backend", "all data models", "all APIs". These deliver a layer, not a feature. Such work belongs in Phase 0 or inside the first slice that needs it.

**Sizing:** one feature or flow per slice — small enough to build, test, and polish in one pass. Two unrelated features → split.

## Phases
Phase 0 is fixed; user (as PM) shapes Phase 1+ order.

0. **Phase 0 — Foundation:** Scaffolding + app shell + full planned UI (polished static, mock data). The whole product, clickable and visually final; nothing wired yet.
1. **Phase 1 — [Feature Name]:** [The one feature this slice brings to life, end-to-end. Why it comes first.]
2. **Phase 2 — [Feature Name]:** [The one feature this slice delivers. Why here.]
3. **Phase 3 — [Feature Name]:** [...]

## Global Out of Scope
What this project will explicitly never do. Named precisely — not vague.

- [Specific exclusion]
- [Specific exclusion]
