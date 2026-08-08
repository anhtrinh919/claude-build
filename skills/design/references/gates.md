# Gate catalogue — the Definition of Done

A gate does not pass by assertion. Each gate returns objective evidence, or it is blocked. Rank every gate by the judgment it needs:

- **Tier 1 — objective (hard-block).** Answer from the screen→image index and the image set by enumeration: present or absent, reachable or orphaned, designed or missing. Paste the enumeration as evidence.
- **Tier 2 — evidence (artifact required).** For what only vision can judge (density, overlap, clipping, contrast, "matches the intent"), point to the exported image and judge it against the gate's check. No image, no pass. Mark the result as an estimate and name what the designer must verify in-tool.

Static exports have no DOM, CSS, or runtime. Deterministic checks (axe, grep, DOM measurement) run at build time; gates below name which checks hand off there.

**Which track runs this.** On `claude-code`, impeccable owns every check below against its own live mockups; this catalogue does not run there. This doc is the external track's catalogue, run against exported static images plus the screen→image index (`handover.md`).

Each gate lists: tier · what it verifies · check · evidence · applicable when.

---

## Structure & navigation — checked against the screen→image index

**G-REACH** — *Tier 1, high value.* No orphan and no dead-end screens.
- **Check:** build the adjacency list from the screen→image index and each image's nav/links. Every screen needs ≥1 inbound path from a reachable start and ≥1 outbound forward action plus a back path, with a stable URL or deep link; a screen reachable only as a modal state is a finding.
- **Evidence:** the adjacency list + a `0 orphans, 0 dead-ends` line.
- **Applies:** always, scoped to screens built so far (`product.md`'s `built`/`changed` rows) — an unbuilt screen isn't an orphan.

**G-STATES** — *Tier 1.* Every data-driven screen ships Loading + Empty + Error + Not-found + Offline.
- **Check:** per screen, list which of the five states are designed; a data screen missing any fails.
- **Evidence:** the per-screen state matrix.
- **Applies:** when any screen shows data (almost always).

**G-NAV** — *Tier 2 (vision estimate).* Every non-home screen has a back affordance; web logo links home; "you are here" is answered.
- **Check:** per screen image, name its back mechanism (nav back / breadcrumb / ancestor link). For web, confirm the header shows a logo/wordmark and ask the designer to confirm it links to `/`.
- **Evidence:** table of `screen → back mechanism`; any "none" fails.
- **Applies:** always (logo→home is web-only).

---

## Flow & IA

**G-SCREEN-JOB** — *Tier 2 (evidence).* Each screen maps to one job + one primary action.
- **Check:** produce a `screen → user job → primary action` inventory; a blank primary action = orphan = fail.
- **Evidence:** the completed table.
- **Applies:** always.

**G-AUTH-SET** — *Tier 1.* Complete set present: sign-up, log-in, log-out, password-reset, plus generic (non-enumerating) login errors.
- **Check:** assert the four screens/states exist in the exported images (per the index) and that the login-error copy shown is generic.
- **Evidence:** checklist of the four + the error string.
- **Applies:** only when the app has accounts.

**G-DESTRUCT** — *Tier 2.* Each destructive action has confirmation/undo, action-labeled buttons, no dangerous default, separated from benign actions.
- **Evidence:** list of destructive actions + their guard.
- **Applies:** only when destructive actions exist.

**G-FORMS** — *Tier 2 (vision estimate).* Inputs have visible, persistent labels; validation fires on blur not keystroke and errors sit next to fields.
- **Check:** per form image, confirm each field shows a visible label distinct from its placeholder (not placeholder-as-label); note validation-timing intent where shown. Programmatic label wiring (`for`/`aria-label`) is a DOM property invisible in a static image — flag it as a build-time check (axe).
- **Evidence:** per-form checklist of visible labels + a one-line note on validation timing.
- **Applies:** when the phase has forms.

---

## Accessibility — checked against the exported images

**G-A11Y-AXE** — *Tier 2 (vision estimate — degraded from Tier 1; axe-core needs live DOM, not available on a flat export).* WCAG basics: text contrast ≥4.5:1 (≥3:1 for large text, ≥18pt or ≥14pt bold); non-text contrast ≥3:1 for UI boundaries, icons, and states; text resizes to 200% without loss; every control exposes name, role, and state, native or via correct ARIA; meaningful images carry `alt`, decorative ones `alt=""`; color never carries information alone — pair it with text, an icon, or a pattern.
- **Check:** per image, eyeball contrast on every text/background and icon/background pair. Flag any control missing a visible name or any image with no apparent alt-equivalent context. Send borderline or failing flags to the designer to verify with axe or browser devtools before build.
- **Evidence:** per-image list of flagged contrast pairs / missing names — or "no flags," each marked "verify with a checker."
- **Applies:** always.

**G-A11Y-TARGET** — *Tier 2 (vision estimate).* Tap targets ≥44px / 48dp.
- **Check:** per image, judge interactive-element size against the image's own scale (a stated screen width or known reference element as ruler); flag anything visibly under the 44px/48dp minimum.
- **Evidence:** list of elements flagged as likely undersized (must be empty, or each flagged item sent back to the designer to verify against spec).
- **Applies:** always.

**G-A11Y-FOCUS** — *Not observable on a static image unless the designer exported it.* Visible focus on all interactive elements, not obscured by sticky chrome. Standard: `:focus-visible`, a ring of 2px or more at ≥3:1 contrast, never `outline:none` without a replacement, and `scroll-margin` so sticky headers do not cover the focused element.
- **Check:** ask whether the design spec calls out a focus treatment, or an element is exported in its focused state. If neither exists, flag it as a gap for the designer.
- **Evidence:** the exported focus-state image or the written focus-treatment spec — or, absent both, the flag asking the designer to add one.
- **Applies:** always; reports `— not observable` (advisory, does not block — rule 2) when neither exists. The build-time a11y check covers it.

**G-A11Y-KBD** — *Mostly unobservable on a static image.* Keyboard-operable: everything interactive works by keyboard alone; native elements (`<button>`, `<a>`) get this free; no `tabindex > 0`; tab order matches visual order; skip-to-main-content link on web.
- **Check:** tab order, native-vs-non-native choice, and tabindex misuse are DOM properties invisible in a flat image — flag as a build-time check (keyboard walkthrough + axe). The one thing an image can show: a visible skip-to-content affordance on a web screen.
- **Evidence:** skip-link present/absent per web screen (or "not shown") + a note the rest is out of scope for this track.
- **Applies:** always for the skip-link check; the rest reports `— not observable` and does not block (rule 2), handed to the build-time keyboard walkthrough.

**G-MOTION** — *Not applicable on external.* `prefers-reduced-motion: reduce` honored; durations 150–300ms.
- **Check:** motion has no static-image signal. Note intent if the brief/handoff states it; otherwise there is nothing to check.
- **Evidence:** the motion-intent note if one exists, or none.
- **Applies:** not applicable on this track — verify at build/code-review time instead.

---

## Visual system & quality

**G-TOKENS** — *Tier 2 (vision estimate).* Color/spacing look drawn from one consistent set, not ad-hoc per screen. The standard is `design-tokens.md`.
- **Check:** across the exported images, does color/spacing look drawn from a consistent, repeated set (a small palette, a spacing scale) or ad-hoc screen to screen?
- **Evidence:** violation list naming the inconsistent values (empty or justified).
- **Applies:** always.

**G-RESPONSIVE** — *Tier 2.* Renders cleanly at mobile / tablet / desktop — no overflow, cutoff, or overlap.
- **Check:** review whichever breakpoint images the designer exported per key screen for overflow, cutoff, or overlap. A screen exported at only one breakpoint is itself a finding — name which breakpoints are missing.
- **Evidence:** per-screen list of breakpoints reviewed + pass/fail note + missing-breakpoint call-outs.
- **Applies:** always. A defect found in an exported breakpoint blocks; missing breakpoints are advisory — report them, don't hold the boundary on them.

**G-DENSITY** — *Tier 2, always.* Each key screen is neither too busy nor too sparse. No region is overloaded (a wall of competing controls, nothing breathing) and none is vacuously empty (one control adrift in dead space). The deeper "is the hierarchy right" call stays the human's at phase-end review.
- **Check:** the exported image per key screen → `busy` / `balanced` / `sparse` verdict + one line naming the worst region. Any `busy`/`sparse` blocks until fixed or justified.
- **Evidence:** the exported image path + the density verdict.
- **Applies:** always.

**G-DEFECTS** — *Tier 2, always.* No concrete rendering defect on any key screen: element overlap/collision, content clipped or cut off, a broken/missing image, an empty or placeholder control that should hold content, or more than one primary CTA at equal visual weight (count them — >1 fails). `G-A11Y-AXE` covers contrast and `G-RESPONSIVE` covers breakpoint overflow; this gate covers the rest.
- **Check:** a vision pass per exported image for overlap/collision, clipping, broken images, empty controls, and equal-weight CTAs → defect list or "0 defects."
- **Evidence:** the exported image path + defect list (or "0 defects").
- **Applies:** always.

**G-REFERENCE** — *Tier 2.* The exported image vs the user's `reference.png` — enumerate every visual difference.
- **Check:** compare the two images directly; list each difference concretely (what, where, how). Every difference goes into `design-comment.md` as a change request; the boundary stays closed until the user returns a revised export.
- **Evidence:** side-by-side + the difference list, carried into `design-comment.md`.
- **Applies:** only when the user supplied a reference image / "make it look like this."

---

## The gate report (what the external design review emits)

A table of `gate · tier · applies · status · evidence` — the detail behind `design-comment.md`. Rules:

1. Run every applicable gate. Scope with "applicable when" — don't block a marketing page on `G-AUTH-SET`.
2. **You may not report design complete while any applicable gate is unknown or failing.** Tier-1 blocks on its output; Tier-2 blocks on a missing artifact. **Not-observable-here is not "unknown":** a gate a static export cannot answer (focus treatment not exported, keyboard order, motion) reports `— not observable on this track`, names the build-time check that covers it, and does **not** block.
3. A non-applicable or not-observable gate is marked `—` with the reason. Never drop it silently — that reads as "covered" when it wasn't.
4. Keep the gate set **few and hard**. Resist adding soft gates. If something keeps slipping through, add one hard check, not ten soft ones.

Green-or-justified across the board → hand back for user approval (/build:design's approval gate).
