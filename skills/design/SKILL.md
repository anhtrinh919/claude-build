---
name: design
description: Design phase of /build. Two modes. **claude-code** hands the UI to `impeccable`, which designs it in-codebase as mockups. **external** writes a brief, waits for the exported images, reviews them against a gate catalogue, writes design-comment.md and a screen→image index. Runs from /build.
---

# /build:design — pick a mode, hand off or review, close out

On **`claude-code`** the work is delegated whole to the **`impeccable` skill**; its mockups live in the repo and **are** the design, so this mode writes no handover doc. On **`external`** the user designs in their own tool; this skill briefs, waits, reviews the exported images, and builds the screen→image index. impeccable cannot run `external` — it needs live code, not a static image.

Enter through `/build` (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`). Plain language to the user, never paths or stack names (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Mode | Model | Mechanism | Reads | Writes | Terminal step |
|---|---|---|---|---|---|
| **claude-code** | Opus | inline — invokes the `impeccable` skill, which drives its own work and holds the user gate | `requirements.md` (the contract) · `product.md` · `outcome-card.md` · prior `design-tokens.css` | `specs/<phase>/mockups/` · `design-tokens.css` · non-visual decisions → `docs/decisions.md`. **No handover doc** | **none** |
| **external** | Opus | inline — fans out a Sonnet leaf to read the exported images and return text findings | same, plus the user's exported images | `design-brief.md` · `design-tokens.css` · `design-comment.md` · `handover.md` (screen→image index) | **none** |

Mode is an arg; absent → resolve it as a fork below. Both hold a live user gate, so both run inline (`_shared/subagent-policy.md` Rule 1).

**No terminal step.** Return without writing `.build-state.json`. The orchestrator runs **design-compliance**: pass → `design-complete`; fail → roll back to `spec-complete` and re-run this skill. Hand straight back (`_shared/auto-continue.md`). No `requirements.md` in a `specs/` dir → stop: "No phase spec found — spec must run first."

**Standalone** (`/build:design` typed directly): skip the build-stack file contracts. `claude-code` → invoke impeccable on the request. `external` → review the images they point at and write the change list. No gate follows, so surface the outcome yourself and stop.

## Mode resolution (first action)

Absent arg → check `docs/decisions.md ## User decisions` for a prior phase's pick. Found `claude-code` → **stays `claude-code` forever, no ask.** Found `external`, or nothing yet → a felt tooling fork, resolved by *"does this phase establish new visual language?"* with one `AskUserQuestion`, recommended first:

- **Phase 0** → recommend **external**: it locks the tokens and visual language, and a dedicated design tool is materially stronger at that founding pass.
- **Phase 1+** → **external** for a new visual vocabulary (new token family, a component category absent from `design-tokens.css`, ~3+ new screens). **`claude-code`** otherwise.

Record the pick to `mission.md ## Design Tool` (/build:review reads it back) and the fork to `docs/decisions.md`.

**North star — once, then carried forward.** `mission.md ## Design North Star` absent → ask one plain question ("what should a user remember or feel after using this?") and write it there. One app, one north star. It is the top-line context for impeccable, or the brief's `## Design intent`.

---

## Mode: claude-code

**Invoke the `impeccable` skill and let it work.** It owns structure, navigation, accessibility, craft, the mockups, the live browser show, and the felt-decision forks. Do not re-derive it, wrap it in a gate scaffold, or restate a craft ruleset.

Hand it: the **north star**; the **screens this phase must cover** (from `requirements.md` — on `phase: 0`, the shell and hero screens from `product.md`'s Phase 0 Foundation Scope, built to final static polish, realistic mock data, pixel-complete, clickable, unwired); the **existing `design-tokens.css`** if a prior phase set one, so it extends the locked language; **2-3 real reference screens for this screen type from the Mobbin MCP**; the stack from `tech-stack.md`. Output home is `specs/<phase>/mockups/`, where backend and the compliance gate expect it.

**Close-out, once impeccable's work is approved:**

1. **`design-tokens.css`** — the mockups' shared token layer at `specs/<phase>/design-tokens.css`, written to the standard in `references/design-tokens.md`: one CSS custom property per token under `:root`, generated-from header. Backend imports this one file. Fonts not yet in the app → note them in the decisions log.
2. **`docs/decisions.md`** — every **non-visual** decision the mockups do not show on their face: a structural choice and its why (slide-over not modal, wizard not one form), any deliberate deviation from `requirements.md`, component→name mappings an implementer might miss. Never restate what is visible in the pixels.

---

## Mode: external

### Step 1 — Design brief

`specs/<phase>/design-brief.md` already exists → the user is returning from their tool. Do not rewrite it; skip to Step 3.

Otherwise write it from `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/design-brief-external.md`. **Include:** product context, mental model, the full spec (screens, states, behaviors, forward-compat callouts, IA, stories), real on-screen copy, the numbered screen checklist. **Exclude everything the design tool owns** — visual direction, palette, typography, mood, component decisions, layout hierarchy, interaction patterns, breakpoints, spacing. Carry the north star into `## Design intent`.

### Step 2 — Hand off and wait

Convey: the brief is written, design it in whatever tool you like, cover every screen on the checklist (paste the numbered list), export the screens as images when happy. Then **stop** — nothing runs in the meantime. Do not design on the user's behalf.

### Step 3 — Review the exported images

Review against `references/gates.md` as written — it is native to static exports, do not re-adapt it. First map every screen in `requirements.md` to its exported image; that mapping *becomes* the index. Then judge each image via a Sonnet leaf returning text findings plus paths — never image bytes into the main session.

- **Structural**, from the index and requirements (deterministic): every required screen present? every data screen's five states (Loading / Empty / Error / Not-found / Offline) designed? every non-home screen has a back affordance? any orphan or dead end? auth set complete where accounts exist? (`G-REACH` `G-STATES` `G-NAV` `G-AUTH-SET` `G-SCREEN-JOB`)
- **Visual**, one vision pass per image (`G-DENSITY`, `G-DEFECTS`, `G-A11Y-TARGET`, contrast): a density verdict (too busy / balanced / too sparse) plus concrete defects — overlap, clipping, broken image, empty control, >1 equal-weight primary CTA. Flag likely contrast failures and undersized targets. Do **not** score hierarchy or taste; that is the user's call on their own design. Contrast off an image is an estimate — flag the pair for a checker, never assert a ratio.
- **Tokens** (`G-TOKENS`): color and spacing drawn from a consistent set, or ad-hoc across screens?
- **Forms and destructive** (`G-FORMS`, `G-DESTRUCT`): labels visible rather than placeholder-as-label? destructive actions guarded?

### Step 4 — Write `design-comment.md`

Write `specs/<phase>/design-comment.md`, addressed to the designer. Each finding is **severity · gate ID · what is wrong · exactly what to change**. Order HIGH → LOW within a screen; screens with the most HIGHs first.

```
# Design review — Phase <N> (external)
**Verdict:** <N HIGH · N MED · N LOW>. <"Blocks backend until HIGH resolved" | "Clean — proceed">

## <Screen name>   ← one section per screen with findings
- **[HIGH · G-STATES]** No empty state for the saved-items list. Add one: icon + "Nothing saved yet" + a "Browse articles" button. Dead-ends otherwise.
- **[MED · G-A11Y-TARGET]** The row overflow "⋯" button looks ~28px; bump to ≥44px hit area.

## Cross-cutting
- **[HIGH · G-NAV]** No back affordance on any detail screen — add a back control on every pushed screen.
```

### Step 5 — Surface, gate, index

Show the verdict line and the HIGH items; the design is theirs and they fix it in-tool. Offer **fix and re-review** (loop Step 3 on the revised images, re-checking only previously-failing screens) or **accept and proceed** (a logged override). **A HIGH blocks the design→backend boundary** until resolved or overridden. MED and LOW are advisory.

**Close-out, on a clean or accepted result:**

1. **`design-tokens.css`** — the values the images imply (palette, spacing, type) as one CSS custom property set under `:root`, to the standard in `references/design-tokens.md`, generated-from header naming source and date. Fonts not in the app → a **Fonts required** note.
2. **`handover.md`** — the screen→image index and nothing decorative, per `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/handover.md`: every screen in `requirements.md` → its exact exported image file (or "not designed" plus reason), the design source path marked loudly as what backend builds from, the tokens import path, fonts required, plus the schema's reusable-components, deviations, and IA sections, and any **spec gap**, which stops backend until the spec updates. Never restate colors, sizes, or spacing.

---

## References

- `references/gates.md` — the gate catalogue **external** runs over exported images: each gate's tier, what it verifies, the evidence it produces.
- `references/design-tokens.md` — the standard `design-tokens.css` must meet. Read by both modes at close-out.
- Brain: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md`, `$AGENT=design`.

## Ground rules

1. **`requirements.md` is the contract.** A design needing something absent from the spec → **stop and name the gap**; never invent past it.
2. **Never settle a felt UI decision with words alone.** `claude-code` shows rendered pixels; `external` names the screen, the gate, and the exact change — never "improve the spacing".
3. **Coverage is the completeness contract.** Every screen reaches the mockups, or the index plus `design-comment.md`.
4. **claude-code is impeccable, whole.** Hand off the context and trust it.
5. On `external`, vision work is a text-only Sonnet leaf returning findings and paths.
6. Not approved, or a HIGH still open, means not done.
