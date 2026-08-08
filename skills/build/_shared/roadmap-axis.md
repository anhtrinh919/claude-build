# Roadmap axis doctrine

Two fixed shapes. **Phase 0 is always Foundation** — scaffold, app shell, and hero screens (home or dashboard plus 1–2 top nav screens), built to polished static with mock data, design-locked, nothing wired. **Phases 1–n are vertical slices** — each one feature, wiring one already-built screen to real data and behavior end-to-end, tested and polished before the next. A slice whose screen Phase 0 did not cover builds that screen to polished static first.

**Slice test:** every Phase 1+ must let the user *do* something new end-to-end after it. Never draft a horizontal phase — "build the backend", "all the APIs". A slice the user cannot act on fails the test; re-slice it. Never thin a slice; split an oversized one instead.

**Choose the axis before you order anything.** This is where roadmaps fail. State the slicing axis in one sentence before drafting, and slice by what the product is and what it can do — the capabilities a user would name out loud. Check the axis against all of these:

- **Banned — the pipeline's own work order.** "Collect input → process → generate → review" is the order *you* work in, not a sequence of things a user can do. It is a horizontal roadmap in vertical clothing.
- **Banned — a ladder of output classes.** "Read-only → editable → multi-user" reduces the product to its output, and reads as a lesser product rather than an earlier one.
- **Banned — a property of one imagined output.** Where the product builds arbitrary things for its users, no lifecycle detail of a single hypothetical artifact can be the axis. That slices a strawman.
- **Every seam must exist in the user's reality.** Before you propose a boundary, name the moment the user would experience it. Two capabilities that are one act on one surface are ONE phase. Splitting them to make the engineering smaller is the most common failure in this step.
- **Failure signal.** If the user says the roadmap "has no logic", or singles out one phase as odd, **the axis is wrong — not that phase.** Reslice from a different axis. Never patch the odd phase and re-present the same shape.
- **When a draft misses twice, explore before you commit.** Fan out 3–4 leaf agents on genuinely distinct axes, each with the same product shape and the rejected axes named, then compare their tables and pick or synthesize. Cheaper than a third rejected draft.

**Deferring phases 1+ is legitimate.** Where the product's final shape is not yet concrete enough to derive a sequence, say so and defer rather than guess. Phase 0 locks the shell and hero screens, which is enough for the roadmap to become derivable.
