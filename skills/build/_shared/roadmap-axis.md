# Roadmap axis doctrine

Two fixed shapes: **Phase 0 is always Foundation** (scaffold + app shell + the full planned UI built to polished static — every screen, mock data, design-locked, nothing wired). **Phases 1–n are vertical slices** — each ONE feature wiring one already-built screen to real data + behavior end-to-end, tested and polished before the next.

**Slice test:** every Phase 1+ must let the user *do* something new end-to-end after it. Never draft a horizontal phase ("build the backend," "all the APIs") — a slice the user can't act on fails the test; re-slice it. Never thin a slice — split an oversized one.

**Choose the axis before you order anything — this is where roadmaps actually fail.** State your slicing axis in one sentence before drafting, and slice by *what the product is and what it can do* — the capabilities a user would name out loud. Check the axis against all of these:
- **Banned — the pipeline's own work order.** "Collect input → process → generate → review" is the order *you* do work in, not a sequence of things a user can do. A horizontal roadmap wearing vertical clothing.
- **Banned — a ladder of output classes.** "Read-only version → editable version → multi-user version" reduces the product to whatever its output happens to be, and reads as a lesser product rather than an earlier one.
- **Banned — a property of one imagined output.** If the product builds arbitrary things for its users, no lifecycle detail of a single hypothetical artifact can be the axis; you'd be slicing a strawman.
- **Every seam must exist in the user's reality.** Before proposing a boundary, name the moment the user would experience it. If two capabilities are one act on one surface, they are ONE phase — splitting them because it makes the engineering smaller is the single most common failure in this step.
- **Failure signal.** If the user says the roadmap "has no logic," or singles out one phase as odd or weird, **the axis is wrong — not that phase.** Reslice from a different axis; never patch the odd phase and re-present the same shape.
- **When a draft misses twice, explore before committing.** Fan out 3-4 leaf agents on genuinely distinct axes (value moment, risk-first, depth of guarantee, free choice) with the same product shape and the rejected axes named, then compare their tables and pick or synthesize. Cheaper than a third rejected draft.

**Deferring phases 1+ is legitimate.** When the product's final shape isn't yet concrete enough to derive a sequence, say so and defer rather than guessing: Phase 0 builds every planned screen to polished static, which *is* the final shape — once it exists the roadmap becomes derivable.
