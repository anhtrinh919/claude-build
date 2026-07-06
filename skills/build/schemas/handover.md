# Phase [N] Frontend Handover — [Feature Name]

**External track only.** This file is the **bare screen→image index** for a design the user made in their own tool and exported as images. (The `claude-code` track writes no handover — its mockups are real code in `specs/<phase>/mockups/` and backend builds from them directly.)

## Design source — the exported images

> **This handover is an index, not a specification.** The exported images ARE the design; backend builds each screen to match its image. This file only says *which image is which screen* — it never restates colors, spacing, or layout (those are in the images + tokens).

- **Images path:** `[folder or list of exported image files]` — **backend builds each screen to match its image.**
- **Tokens file:** `specs/YYYY-MM-DD-[feature]/design-tokens.css` — import from the app's global stylesheet; reference via CSS variables (`var(--surface-primary)`), never duplicate hex/px into component styles.

## Fonts

Every font family the design uses. Backend configures loading once (e.g. `next/font/google` or local `@font-face`) and wires the variables into `design-tokens.css`.

| Role | Family | Weights used |
|------|--------|--------------|
| Headings | [e.g. Newsreader] | 400, 500 |
| Body | [e.g. Inter] | 400, 500 |
| Monospace | [e.g. Geist Mono] | 400 |

## Screen → image index

The mapping from each requirement-spec state to the exact exported image. Backend opens the named image to build that state.

| Requirement state | Image file | Notes |
|-------------------|------------|-------|
| Home — empty | `home-empty.png` | |
| Home — populated | `home.png` | |
| Board — running card | `board-running.png` | |
| ... | ... | |

Every state listed in `requirements.md` UI Requirements must appear here. A state intentionally not designed → `— not designed —` + reason.

## Reusable components

Visual elements that repeat across screens (e.g. "Phase Card / Running"). Backend implements each as one reusable component with state variants — never inline-duplicated per screen.

| Component name | Appears in screens |
|----------------|--------------------|
| [e.g. Phase Card] | Board, Card Detail |
| ... | ... |

## Deviations from requirements spec

Any design that diverges from `requirements.md`. What and why.

- **[What deviated]:** [Why] — **Impact on backend:** [None / describe]

*No deviations — design matches spec exactly.* [Delete if there are deviations]

## Layout / IA notes

Only structural patterns an implementer might miss by pattern-matching existing code (e.g. "No left sidebar — top-header throughout." "Card detail is a right-side sheet, not a modal."). Never restate what the images show.

*No IA deviations — design follows existing app structure.* [Delete if there are notes]

## Spec gaps (if any)

> ⚠ **Spec gap:** the design needs `[screen/state/endpoint]` that `requirements.md` does not cover. Do not start backend until the spec is updated.

*No gaps — the design covers exactly the spec's screens.* [Delete if there are gaps]
