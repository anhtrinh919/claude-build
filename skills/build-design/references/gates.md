# Gate catalogue — the Definition of Done

A gate is only as strong as its verifiability. The agent cannot pass a gate by assertion — each returns **objective evidence or is blocked**. Rank every gate by how little judgment it needs:

- **Tier 1 — deterministic (hard-block).** A script/scanner/grep returns pass/fail; the agent pastes the output. You cannot fake `axe: 0 violations` or a wireflow with an orphan node.
- **Tier 2 — evidence (artifact required).** For what only vision can judge (density, overlap, clipping, "matches the intent"), the gate is not "did you check" — it's "save the screenshot to `.review/` and judge it against the named gate's check." No artifact → not done.

**Which track runs this.** On `claude-code`, **impeccable** owns every check below against its own live mockups — this catalogue is not invoked there. This doc is the **external** track's review catalogue: the "how to check" steps below are written against rendered HTML (their original claude-code form); on external you **adapt each to the exported static image** — the deterministic grep/axe checks become per-image vision estimates (see each gate's own external note, and G-A11Y), the structural gates run against the screen→image index as the wireflow. Where a check reads "the mockup," read "the exported image" on this track.

Each gate lists: **tier · what it verifies · how to check (existing tooling only) · evidence · applicable when.** IDs map to rule IDs in [ux-rules.md](ux-rules.md).

---

## Structure & navigation — checked against the wireflow (Milestone 1)

**G-REACH** — *Tier 1, high value.* No orphan and no dead-end screens (N10, F3, F6).
- **Check:** over the Milestone-1 screen graph, assert every screen has ≥1 inbound path from a reachable start AND ≥1 outbound forward action + a back path.
- **Evidence:** the adjacency list + a `0 orphans, 0 dead-ends` line, and the rendered wireflow PNG.
- **Applies:** always.

**G-STATES** — *Tier 1.* Every data-driven screen ships Loading + Empty + Error + Not-found + Offline (N13–N17).
- **Check:** per screen on the map, list which of the five states are designed; a data screen missing any fails.
- **Evidence:** the per-screen state matrix.
- **Applies:** when any screen shows data (almost always).

**G-NAV** — *Tier 1.* Every non-home screen has a back affordance; web logo → home; "you are here" is answered (N1, N2, N6).
- **Check:** per screen, name its back mechanism (nav back / breadcrumb / ancestor link); grep the mockup header for a logo anchor to `/`.
- **Evidence:** table of `screen → back mechanism`; any "none" fails.
- **Applies:** always (logo→home is web-only).

---

## Flow & IA

**G-SCREEN-JOB** — *Tier 2 (evidence).* Each screen maps to one job + one primary action (F1–F3).
- **Check:** produce a `screen → user job → single primary action` inventory; a blank primary action = orphan = fail.
- **Evidence:** the completed table.
- **Applies:** always.

**G-AUTH-SET** — *Tier 1.* Complete set present: sign-up, log-in, log-out, password-reset, plus generic (non-enumerating) login errors (F13–F20).
- **Check:** assert the four screens/states exist in the mockups and that the login-error copy is generic.
- **Evidence:** checklist of the four + the error string.
- **Applies:** only when the app has accounts.

**G-DESTRUCT** — *Tier 2.* Each destructive action has confirmation and/or undo, action-labeled buttons, no dangerous default, separated from benign actions (F24–F28).
- **Evidence:** list of destructive actions + their guard.
- **Applies:** only when destructive actions exist.

**G-FORMS** — *Tier 1 + 2.* Inputs have programmatic labels (Tier 1, axe catches missing labels); validation fires on blur not keystroke and errors sit next to fields (Tier 2, evidence) (F29–F34).
- **Evidence:** axe label output + a one-line note on validation timing per form.
- **Applies:** when the phase has forms.

---

## Accessibility — deterministic, run against the mockup HTML

**G-A11Y-AXE** — *Tier 1, hard-block.* axe-core passes: contrast ≥4.5:1 (3:1 large), name/role/value, ARIA correctness, alt text.
- **Check:** runnable code only — load each mockup in `/browse`, inject axe-core, run `axe.run()`, collect violations. On the `claude-code` track impeccable owns this against its live mockups; on `external` (static exported images) axe can't run — estimate contrast/name-role-value per image and tell the designer to verify with a checker, don't assert a false-precise result.
- **Evidence:** axe summary — **0 violations** (or documented, justified exceptions) + saved `.review/axe.json`.
- **Applies:** always.

**G-A11Y-TARGET** — *Tier 1.* Tap targets ≥44px / 48dp (A1).
- **Check:** axe target-size rule, or a one-off DOM-measurement snippet over interactive elements.
- **Evidence:** list of undersized nodes (must be empty).
- **Applies:** always.

**G-A11Y-FOCUS** — *Tier 1.* Visible focus on all interactive elements; not obscured by sticky chrome (A2–A3).
- **Check:** grep the mockup CSS for `:focus-visible` styling and the absence of a blanket `outline:none` without replacement; axe focus rules.
- **Evidence:** the focus-style rule + axe output.
- **Applies:** always.

**G-A11Y-KBD** — *Tier 1.* Keyboard-operable: native elements, no `tabindex>0`, skip link on web (A4–A5).
- **Check:** grep for `tabindex="[1-9]`, non-native clickable `div`/`span` handlers, and a skip link.
- **Evidence:** grep results (empty = pass).
- **Applies:** always.

**G-MOTION** — *Tier 1.* `prefers-reduced-motion: reduce` honored; durations 150–300ms (A8).
- **Check:** grep for the media query wrapping motion and transition durations.
- **Evidence:** the matched block.
- **Applies:** only when animations exist.

---

## Visual system & quality

**G-TOKENS** — *Tier 1.* No hardcoded colors/spacing in component markup — values come from tokens (ux-rules §4).
- **Check:** grep the mockup component files for raw hex (`#[0-9a-fA-F]{3,6}`) or arbitrary px outside the token/theme layer.
- **Evidence:** violation list (empty or justified).
- **Applies:** always for the Claude-Code track (we author the tokens).

**G-RESPONSIVE** — *Tier 2.* Renders cleanly at mobile / tablet / desktop — no overflow, cutoff, or overlap (A7).
- **Check:** screenshot each key screen at 390 / 768 / 1280px; assert `scrollWidth ≤ clientWidth` at 390 (no horizontal overflow).
- **Evidence:** the 3-width screenshots in `.review/` + a pass note.
- **Applies:** always.

**G-DENSITY** — *Tier 2, always.* Each key screen is neither too busy nor too sparse — density earns its place. No region is overloaded (a wall of competing controls, nothing breathing) and none is vacuously empty (one control adrift in dead space, unused viewport). This is the clutter/minimalism call — the one holistic-ish judgment a model makes reliably. The deeper "is the hierarchy right / what do I look at first" call is **not** gated here — it stays the human's at phase-end review.
- **Check:** design-time screenshot per key screen → `busy` / `balanced` / `sparse` verdict + one line naming the worst region. Any `busy`/`sparse` blocks until fixed or justified.
- **Evidence:** `.review/<screen>.png` + the density verdict.
- **Applies:** always, both tracks (Claude-Code renders its mockup; external judges the exported image).

**G-DEFECTS** — *Tier 1 + 2, always.* No concrete rendering defect on any key screen: element overlap/collision, content clipped or cut off at the default width, a broken/missing image, an empty or placeholder control that should hold content, or more than one primary CTA at equal visual weight (count them — >1 fails). Contrast and programmatic labels are already hard-gated by `G-A11Y-AXE`, and breakpoint overflow by `G-RESPONSIVE` — this gate is the concrete defects those two don't cover. These are the defects a model detects reliably (~80–85%); they are flags, not a score.
- **Check:** design-time screenshot per key screen + light DOM checks (overlap via overlapping bounding boxes; count elements styled as a primary CTA) → defect list or "0 defects."
- **Evidence:** `.review/<screen>.png` + defect list (or "0 defects").
- **Applies:** always, both tracks.

**G-REFERENCE** — *Tier 2.* Screenshot vs a `reference.png` — enumerate every visual difference, fix, repeat until they match.
- **Evidence:** side-by-side + the closed difference list.
- **Applies:** only when the user supplied a reference image / "make it look like this."

---

## The gate report (what the external design review emits)

A table of `gate · tier · applies · status · evidence` — the backing detail behind `design-comment.md`. Rules:

1. Run **every applicable** gate. Scope with "applicable when" — don't block a marketing page on `G-AUTH-SET`.
2. **You may not report design complete while any applicable gate is unknown or failing.** Tier-1 blocks on its output; Tier-2 blocks on a missing artifact.
3. A non-applicable gate is marked `—` with the reason. Never silently dropped — that reads as "covered" when it wasn't.
4. Keep the gate set **few and hard**. This catalogue is already the short strong list; resist adding soft gates. If something keeps slipping through, add one hard check, don't add ten soft ones.

Green-or-justified across the board → hand back for user approval (build-design's approval gate).
