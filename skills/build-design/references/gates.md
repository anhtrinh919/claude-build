# Gate catalogue — the Definition of Done

A gate is only as strong as its verifiability. The agent cannot pass a gate by assertion — each returns **objective evidence or is blocked**. Rank every gate by how little judgment it needs:

- **Tier 1 — objective (hard-block).** Answerable from the screen→image index and the image set by enumeration, not judgment: a screen is present or absent, reachable or orphaned, a state is designed or missing. The agent pastes the enumeration. You cannot fake an index with an orphan node.
- **Tier 2 — evidence (artifact required).** For what only vision can judge (density, overlap, clipping, contrast, "matches the intent"), the gate is not "did you check" — it's "point to the exported image and judge it against the named gate's check." No image → not done. A Tier-2 estimate is an estimate: say so, and tell the designer what to verify in-tool.

Nothing here runs a scanner. Static exports have no DOM, no CSS, and no runtime — the deterministic tooling checks (axe, grep, DOM measurement) belong to build time, and several gates below name what they hand off there.

**Which track runs this.** On `claude-code`, **impeccable** owns every check below against its own live mockups — this catalogue is not invoked there. This doc is the **external** track's review catalogue, exclusively: every check below is written natively against a set of exported static images plus the screen→image index (`handover.md`).

Each gate lists: **tier · what it verifies · how to check · evidence · applicable when.** IDs map to rule IDs in [ux-rules.md](ux-rules.md).

---

## Structure & navigation — checked against the screen→image index

**G-REACH** — *Tier 1, high value.* No orphan and no dead-end screens (N10, F3, F6).
- **Check:** from the screen→image index and each image's visible nav/links, build the adjacency list: does every screen have ≥1 inbound path from a reachable start AND ≥1 outbound forward action + a back path?
- **Evidence:** the adjacency list + a `0 orphans, 0 dead-ends` line.
- **Applies:** always.

**G-STATES** — *Tier 1.* Every data-driven screen ships Loading + Empty + Error + Not-found + Offline (N13–N17).
- **Check:** per screen on the map, list which of the five states are designed; a data screen missing any fails.
- **Evidence:** the per-screen state matrix.
- **Applies:** when any screen shows data (almost always).

**G-NAV** — *Tier 2 (vision estimate).* Every non-home screen has a back affordance; web logo → home; "you are here" is answered (N1, N2, N6).
- **Check:** per screen's exported image, name its back mechanism (nav back / breadcrumb / ancestor link); for web, confirm the header shows a logo/wordmark and check the brief or ask the designer that it links to `/` — a static image can't show the link target itself.
- **Evidence:** table of `screen → back mechanism`; any "none" fails.
- **Applies:** always (logo→home is web-only).

---

## Flow & IA

**G-SCREEN-JOB** — *Tier 2 (evidence).* Each screen maps to one job + one primary action (F1–F3).
- **Check:** produce a `screen → user job → single primary action` inventory; a blank primary action = orphan = fail.
- **Evidence:** the completed table.
- **Applies:** always.

**G-AUTH-SET** — *Tier 1.* Complete set present: sign-up, log-in, log-out, password-reset, plus generic (non-enumerating) login errors (F13–F20).
- **Check:** assert the four screens/states exist in the exported images (per the screen→image index) and that the login-error copy shown is generic.
- **Evidence:** checklist of the four + the error string.
- **Applies:** only when the app has accounts.

**G-DESTRUCT** — *Tier 2.* Each destructive action has confirmation and/or undo, action-labeled buttons, no dangerous default, separated from benign actions (F24–F28).
- **Evidence:** list of destructive actions + their guard.
- **Applies:** only when destructive actions exist.

**G-FORMS** — *Tier 2 (vision estimate).* Inputs have visible, persistent labels; validation fires on blur not keystroke and errors sit next to fields (F29–F34).
- **Check:** per form image, confirm each field shows a visible label distinct from its placeholder (not placeholder-as-label); note validation-timing intent where the brief/annotations show it. Programmatic label correctness (`for`/`aria-label` wiring) is a DOM property invisible in a static image — flag it as a build-time check (axe), don't assert it here.
- **Evidence:** per-form checklist of visible labels + a one-line note on validation timing.
- **Applies:** when the phase has forms.

---

## Accessibility — checked against the exported images

**G-A11Y-AXE** — *Tier 2 (vision estimate — degraded from Tier 1; axe-core needs live DOM, unavailable on a flat export).* Contrast ≥4.5:1 (3:1 large), name/role/value, ARIA correctness, alt text.
- **Check:** per image, eyeball contrast on every text/background and icon/background pair, and note any control that looks like it's missing a visible name or an image with no apparent alt-equivalent context. Flag anything borderline or failing and tell the designer to verify with a real checker (axe, browser devtools) before build — don't assert a precise ratio you didn't measure.
- **Evidence:** per-image list of flagged contrast pairs / missing names — or "no flags," each marked "verify with a checker."
- **Applies:** always.

**G-A11Y-TARGET** — *Tier 2 (vision estimate).* Tap targets ≥44px / 48dp (A1).
- **Check:** per image, judge interactive-element size against the image's own scale (use a stated screen width or known reference element as a ruler) and flag anything that looks visibly under the 44px/48dp minimum.
- **Evidence:** list of elements flagged as likely undersized (must be empty, or each flagged item sent back to the designer to verify against spec).
- **Applies:** always.

**G-A11Y-FOCUS** — *Not observable on a static image unless the designer exported it.* Visible focus on all interactive elements; not obscured by sticky chrome (A2–A3).
- **Check:** a focus state only exists to review if the designer exported one. Ask: did the design spec call out a focus treatment, or is at least one interactive element exported in its focused state? If neither exists, this gate can't be checked from the images — flag it as a gap for the designer to fill, don't silently pass it.
- **Evidence:** the exported focus-state image or the written focus-treatment spec — or, absent both, the flag asking the designer to add one.
- **Applies:** always, but reports `— not observable` (advisory flag, does not block — rule 2) when the designer exported neither; the build-time a11y check covers it.

**G-A11Y-KBD** — *Mostly unobservable on a static image.* Keyboard-operable: native elements, no `tabindex>0`, skip link on web (A4–A5).
- **Check:** tab order, native-vs-non-native element choice, and tabindex misuse are DOM properties invisible in a flat image — don't attempt to infer them here; flag them as a build-time check (keyboard walkthrough + axe). The one thing an image can show: if the designer included a visible skip-to-content affordance on a web screen, confirm it's present.
- **Evidence:** skip-link present/absent per web screen (or "not shown") + an explicit note that the rest of this gate is out of scope for this track.
- **Applies:** always for the skip-link check; the rest reports `— not observable` and does not block (rule 2), handed to the build-time keyboard walkthrough.

**G-MOTION** — *Not applicable on external.* `prefers-reduced-motion: reduce` honored; durations 150–300ms (A8).
- **Check:** motion has no static-image signal at all. If the brief/handoff states motion intent (what animates, rough duration, reduced-motion fallback), note it; otherwise there's nothing to check.
- **Evidence:** the motion-intent note if one exists, or none.
- **Applies:** not applicable on this track — verify at build/code-review time instead.

---

## Visual system & quality

**G-TOKENS** — *Tier 2 (vision estimate).* Color/spacing look drawn from one consistent set, not ad-hoc per screen (ux-rules §4).
- **Check:** across the exported images, does color/spacing look drawn from a consistent, repeated set (a small palette, a spacing scale) or ad-hoc/inconsistent screen to screen?
- **Evidence:** violation list naming the inconsistent values (empty or justified).
- **Applies:** always.

**G-RESPONSIVE** — *Tier 2.* Renders cleanly at mobile / tablet / desktop — no overflow, cutoff, or overlap (A7).
- **Check:** review whichever breakpoint images the designer actually exported per key screen for overflow, cutoff, or overlap. A screen exported at only one breakpoint is itself a finding — name which breakpoints are missing and ask the designer to export them.
- **Evidence:** per-screen list of breakpoints reviewed + pass/fail note + missing-breakpoint call-outs.
- **Applies:** always. A defect *found* in an exported breakpoint blocks; **missing** breakpoints are advisory — one export per screen is the common case, and the responsive behaviour gets its real check at build time. Report them, don't hold the boundary on them.

**G-DENSITY** — *Tier 2, always.* Each key screen is neither too busy nor too sparse — density earns its place. No region is overloaded (a wall of competing controls, nothing breathing) and none is vacuously empty (one control adrift in dead space, unused viewport). This is the clutter/minimalism call — the one holistic-ish judgment a model makes reliably. The deeper "is the hierarchy right / what do I look at first" call is **not** gated here — it stays the human's at phase-end review.
- **Check:** the exported image per key screen → `busy` / `balanced` / `sparse` verdict + one line naming the worst region. Any `busy`/`sparse` blocks until fixed or justified.
- **Evidence:** the exported image path + the density verdict.
- **Applies:** always.

**G-DEFECTS** — *Tier 2, always.* No concrete rendering defect on any key screen: element overlap/collision, content clipped or cut off, a broken/missing image, an empty or placeholder control that should hold content, or more than one primary CTA at equal visual weight (count them — >1 fails). Contrast is already estimated by `G-A11Y-AXE`, and breakpoint overflow by `G-RESPONSIVE` — this gate is the concrete defects those two don't cover. These are the defects a model detects reliably (~80–85%) by eye; they are flags, not a score.
- **Check:** a pure vision pass per exported image — no DOM measurement is possible on a flat image — for overlap/collision, clipping, broken images, empty controls, and equal-weight CTAs → defect list or "0 defects."
- **Evidence:** the exported image path + defect list (or "0 defects").
- **Applies:** always.

**G-REFERENCE** — *Tier 2.* The exported image vs the user's `reference.png` — enumerate every visual difference.
- **Check:** compare the two images directly; list each difference concretely (what, where, how it differs). You do not fix these — the design is the user's, authored in their tool; every difference goes into `design-comment.md` as a change request and the boundary stays closed until they return a revised export.
- **Evidence:** side-by-side + the difference list, carried into `design-comment.md`.
- **Applies:** only when the user supplied a reference image / "make it look like this."

---

## The gate report (what the external design review emits)

A table of `gate · tier · applies · status · evidence` — the backing detail behind `design-comment.md`. Rules:

1. Run **every applicable** gate. Scope with "applicable when" — don't block a marketing page on `G-AUTH-SET`.
2. **You may not report design complete while any applicable gate is unknown or failing.** Tier-1 blocks on its output; Tier-2 blocks on a missing artifact. **Not-observable-here is not "unknown":** a gate a static export physically cannot answer (focus treatment the designer didn't export, keyboard order, motion) reports `— not observable on this track`, names the build-time check that will cover it, and does **not** block. Blocking on it would block every external design forever.
3. A non-applicable or not-observable gate is marked `—` with the reason. Never silently dropped — that reads as "covered" when it wasn't.
4. Keep the gate set **few and hard**. This catalogue is already the short strong list; resist adding soft gates. If something keeps slipping through, add one hard check, don't add ten soft ones.

Green-or-justified across the board → hand back for user approval (build-design's approval gate).
