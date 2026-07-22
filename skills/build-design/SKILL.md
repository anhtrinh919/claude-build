---
name: build-design
description: >
  The design phase of the /build SDD stack. Picks one of two tracks. **claude-code** — hands the phase's UI to the `impeccable` skill and lets it work: impeccable designs the interface in-codebase as static mockups, owns structure, accessibility, craft, and the live show-and-fork with the user, all of it. This skill only frames the hand-off and closes out (writes design-tokens.css + non-visual decisions so backend can build; the mockups ARE the design, no handover doc). **external** — Claude writes a tool-agnostic design brief, hands it to the user, and waits; when the user returns with their exported design images, Claude reviews them against a UX/a11y gate catalogue (references/gates.md), writes design-comment.md (the change list), and leaves a bare screen→image index so backend knows which picture is which screen. Reads requirements.md; writes design-brief.md + design-tokens.css (+ mockups/ on claude-code; + design-comment.md + a screen→image index on external). Runs inline. Invoked by /build at the design step, or standalone via /build-design.
---

> **Part of `/build`.** On a resume with an active `.build-state.json`, enter through the `/build` orchestrator — it routes here; don't drive this skill off the state file directly (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`, which also lists the standalone-invocable modes).

# build-design — pick a track, hand off (claude-code) or review (external), close out

This skill owns the phase's UI on one of two tracks:

- **`claude-code`** — Claude designs the interface in-codebase, no external tool. The design *work* is delegated whole to the **`impeccable` skill**: it owns structure, accessibility, craft, the mockups, and the live show-and-fork with the user. This skill just hands off to it and closes out. The mockups live in the repo and **are** the design; backend builds from them, so this track writes **no handover document**.
- **`external`** — the user drives the design in whatever tool they like. This skill writes the brief, waits, then — when the user returns with exported images — **reviews the design against a UX/a11y gate catalogue** (`references/gates.md`), writes `design-comment.md` (the precise change list), and builds a bare **screen→image index** so backend can find each screen. impeccable can't run this track (it needs live code, not a submitted static image).

The **app-shell spec** (`references/app-shell-spec.md`) is not a runtime mode of this skill — it's a static reference that **build-spec** reads to fill `requirements.md`'s App-Shell section. This skill never invokes it.

> **Judgment, not lookup.** The instructions fix *contracts* — what must be true, what must be handed off, what the review must contain. How you phrase things to the user is yours. User-facing text is plain language, never file paths or stack names (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Field | Value |
|---|---|
| **Model** | Opus. |
| **Mechanism** | **Inline.** This skill holds a live user gate (the design approval) — a subagent can't (`_shared/subagent-policy.md` Rule 1). On `claude-code` it invokes the `impeccable` skill, which drives its own work. On `external` it fans out a Sonnet leaf to read the user's exported images and return text findings. |
| **Inputs** | `track` ∈ `claude-code` \| `external` (arg; absent → resolve as a fork). `requirements.md` (the contract). `product.md` (screen inventory + Design North Star). `outcome-card.md` (reference only). `design-tokens.css` if a prior phase set it. |
| **Outputs** | `design-brief.md` (external always; claude-code only if impeccable wants one). `design-tokens.css` (always). **`claude-code`:** `specs/<phase>/mockups/`; non-visual decisions logged to `docs/decisions.md`; **no handover doc.** **`external`:** the user's exported images + a bare screen→image index (`handover.md`) + `design-comment.md`. |
| **Terminal step** | **NONE.** Return without writing `.build-state.json`. The orchestrator runs the **design-compliance** gate: pass → it writes `design-complete`; fail → it rolls `step` back to `spec-complete` and re-runs this skill. On return, do NOT yield or summarize-and-stop — hand straight back so the orchestrator auto-continues in the same turn (`_shared/auto-continue.md`). |

`requirements.md` is the contract. No `requirements.md` in a `specs/` dir → stop: "No phase spec found — spec must run first."

**Standalone use** (`/build-design` typed directly, no active build): skip the build-stack file contracts. `claude-code` → invoke impeccable on the user's request and let it work; `external` → point it at the exported images to review and write the change list. No orchestrator gate follows, so surface the outcome to the user yourself and stop.

## The one rule (both tracks obey their half)

**Never settle a UI decision the user will see or feel with words alone.** On `claude-code` that is impeccable's discipline — it shows the user rendered pixels, not descriptions, and forks the felt calls. On `external` it is *evidence per finding* — every `design-comment.md` line names the screen and the exact change, never a vague "improve the spacing."

---

## Track resolution (first action)

Track is an explicit arg. Absent → it's a **felt tooling fork** — resolve it, don't guess. Check `docs/decisions.md ## User decisions` first (a settled track is honored, not re-asked). Otherwise recommend by the principle *"does this phase establish new visual language?"*, then one `AskUserQuestion`, recommended option first:

- **Phase 0 (Foundation)** → recommend **external** — it designs every screen and locks the tokens + visual language; a dedicated design tool is materially stronger at the whole-product visual pass.
- **Phase 1+** → **external** if the phase adds a *new visual vocabulary* (new token family, a component category not in `design-tokens.css`, ~3+ new screens); **`claude-code`** otherwise (new elements within existing tokens, on existing/near-clone screens).

Record the chosen track to `mission.md ## Design Tool` (`build-review` reads it back to pick compliance rules) and the fork to `docs/decisions.md`.

**North star — once, then carry forward.** `mission.md ## Design North Star` absent → ask one plain question ("what should a user remember or feel after using this?"), write it there. Present → read it. One app, one north star — it's the top-line context you hand impeccable (claude-code) or carry into the brief's `## Design intent` (external).

---

## Track: `claude-code` — hand it to impeccable, close out

**Invoke the `impeccable` skill and let it work.** It is a complete design skill — it owns structure, navigation, accessibility, craft (register, color, type, layout, motion, anti-slop), the mockups, the live browser show, and the felt-decision forks with the user. Do not re-derive any of that here, do not wrap it in a milestone/gate scaffold, do not restate a craft or structural ruleset — hand it the context and trust it.

Hand impeccable: the **north star**, the **screens this phase must cover** (from `requirements.md`; on `phase: 0` that's the *whole* `product.md` Screen Inventory to final static polish — realistic mock data, pixel-complete, clickable, unwired), the **existing `design-tokens.css`** if a prior phase set one (so it extends the locked language, doesn't reinvent it), and the project's **actual stack** (`tech-stack.md`). Point it at `specs/<phase>/mockups/` as the output home so backend and the design-compliance gate find the mockups where they expect. Let impeccable run its own process to approval with the user.

**Spec gap.** If the design needs an endpoint or screen not in `requirements.md`, **stop and name it** ("design needs X, not in the spec — the spec must update before backend"); never invent past it.

**Close-out (claude-code), after impeccable's work is approved:**
1. **`design-tokens.css`** — write the mockups' shared token layer to `specs/<phase>/design-tokens.css` (one CSS custom property per token under `:root`, a generated-from header). Backend imports this one file so tokens aren't re-extracted per component. Fonts not yet in the app → note them in the decisions log so backend wires them.
2. **`docs/decisions.md`** — log every **non-visual** decision the mockups don't show on their face: a structural choice and its why (slide-over not modal, wizard not one form), any deliberate deviation from `requirements.md`, component→name mappings an implementer might miss. What IS visible in the pixels (colors, spacing, layout) is never restated here.

Backend reorients from: the mockups in `specs/<phase>/mockups/` (the design), `requirements.md` (the screen list to map them against), `design-tokens.css`, and `docs/decisions.md`. That set is complete with no handover doc.

---

## Track: `external` — brief, wait, review, gate, index

### Step 1 — external design brief (unconditional)

Write `specs/<phase>/design-brief.md` from `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/design-brief-external.md`. **Include:** product context, product type & mental model, full spec (screens, states, behaviors, forward-compat callouts, IA, user stories), real on-screen copy, the numbered screen checklist. **Exclude — the design tool owns the entire look AND layout:** any styling (visual direction, palette, typography, mood, theme), component decisions, layout hierarchy, interaction patterns, breakpoints, spacing. Carry the north star into `## Design intent`.

### Step 2 — hand off and wait

Convey: the brief is written; design it in whatever tool you like, cover every screen on the checklist (paste the numbered list), and **export the screens as images when you're happy.** Then **stop** — do not proceed until the user returns with their images. There is **no background build** and nothing runs in the meantime. Do not design on the user's behalf.

### Step 3 — review the exported design against the gates

The user returns with exported images → review them against `references/gates.md`, adapted to a non-runnable design (the split shifts toward evidence/vision; the catalogue is identical). First map every screen in `requirements.md` (the coverage contract) to its exported image — this mapping *becomes* the screen→image index. Then judge each image via a Sonnet leaf that reads the image files and returns text findings + paths (never image bytes into the main session):

- **Structural — from the index + requirements** (deterministic): every required screen present? every data screen's five states (Loading/Empty/Error/Not-found/Offline) designed? every non-home screen has a back affordance? any orphan or dead-end? auth set complete where accounts exist? (`G-REACH` `G-STATES` `G-NAV` `G-AUTH-SET` `G-SCREEN-JOB`.)
- **Visual — vision pass per image** (`G-DENSITY`, `G-DEFECTS`, `G-A11Y-TARGET`, contrast): give each image a density verdict (too busy / balanced / too sparse) and a concrete-defect list (overlap/collision · clipping/cut-off · broken image · empty control · >1 equal-weight primary CTA); flag *likely* contrast failures and undersized targets. Do **not** score holistic hierarchy or taste — that stays the user's call on their own design. Contrast off an image is an estimate; flag the pair and tell the designer to verify with a checker, don't assert a false-precise ratio.
- **Tokens** (`G-TOKENS`): does color/spacing look drawn from a consistent set, or ad-hoc across screens?
- **Forms / destructive** (`G-FORMS`, `G-DESTRUCT`): labels visible (not placeholder-as-label)? destructive actions guarded, not defaulted?

### Step 4 — write `design-comment.md` (verbatim schema)

Write `specs/<phase>/design-comment.md`, addressed to the designer (the user). Each finding = **severity · gate ID · what's wrong · exactly what to change** — concrete enough to act on without a follow-up question. Order HIGH → LOW within each screen; screens with the most HIGHs first.

```
# Design review — Phase <N> (external)
**Verdict:** <N HIGH · N MED · N LOW>. <"Blocks backend until HIGH resolved" | "Clean — proceed">

## <Screen name>   ← one section per screen with findings
- **[HIGH · G-STATES]** No empty state for the saved-items list. Add an empty screen: icon + "Nothing saved yet" + a "Browse articles" button (the CTA that fills it). Dead-ends otherwise.
- **[MED · G-A11Y-TARGET]** The row overflow "⋯" button looks ~28px; bump to ≥44px hit area.
- **[LOW · G-DEFECTS]** Card gap is 12/20px inconsistently across the grid; settle on the 8pt step.

## Cross-cutting
- **[HIGH · G-NAV]** No back affordance on any detail screen — add a back control on every pushed screen.
```

### Step 5 — surface, gate, index

Show the user the verdict line + the HIGH items (the design is theirs — they fix it in-tool). Offer **fix & re-review** (loop Step 3 on the revised images, re-checking only the previously-failing screens) or **accept & proceed** (a logged override). **A HIGH finding blocks the frontend→backend boundary** until resolved or explicitly overridden. MED/LOW are advisory. On a clean or accepted result → close out (below) and return.

**Close-out (external):**
1. **`design-tokens.css`** — the token file backend imports. Pull the design-system values the images imply (palette, spacing, type) into one CSS custom property set under `:root`, a generated-from header (source + date). Fonts not in the app → a **Fonts required** note so backend wires them.
2. **`handover.md`** — a **bare screen→image index only**, per `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/handover.md`. Backend can't read an image the way it reads a mockup, so it needs the map: every screen in `requirements.md` → its exact exported image file (or "not designed" + reason), plus the design source path (marked loud "backend builds from these images"), the tokens import path, fonts required, and any structural note an implementer might miss. **Not a narration** — no restating colors/sizes/spacing; just the map.

---

## References & schemas (load as needed)

- **Craft, structure, a11y, the mockups, the live show-and-fork on `claude-code`: the `impeccable` skill owns all of it.** Not a reference file here — invoke it and let it work.
- `references/gates.md` — the UX/a11y gate catalogue the **`external`** track runs over the exported images: each gate's tier, what it verifies, the evidence it must produce.
- `references/ux-rules.md` — the structural UX ruleset (navigation, no-dead-ends, flows/IA, the complete auth set, forms, a11y floors) behind the external track's structural gates.
- `references/app-shell-spec.md` — the standard app-shell spec **build-spec** reads to fill `requirements.md`'s App-Shell section. This skill does not invoke it.
- Schemas: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/{design-brief-external,handover}.md` (`handover.md` = the external track's screen→image index only). Voice: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`. Brain: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/brain.md`, `$AGENT=design`.

---

## Ground rules

- `requirements.md` is the contract; the design serves it. A design that needs something not in the spec → **stop** and surface the gap; never invent past it.
- The coverage checklist is the completeness contract — every screen reaches the mockups (claude-code) or the screen→image index + `design-comment.md` (external).
- **claude-code = impeccable, whole.** Don't wrap it in a milestone/gate scaffold or restate a craft/structural ruleset — hand off the context and trust it. **Evidence per finding** (`external`): every `design-comment.md` line names the screen, the gate, and the exact change.
- On `external`, rendering/vision on submitted images is a text-only Sonnet leaf (reads image files, returns findings + paths) — no browser/test framework in the main session.
- **No handover doc on `claude-code`** — the mockups are the design; backend reads them + `requirements.md` + `docs/decisions.md`. `external` writes only the bare screen→image index.
- No terminal `step` — return; the orchestrator gates design-compliance and writes `design-complete` (or rolls back to `spec-complete` and re-runs this skill).
- Not approved / a HIGH still open = not done. `external` keeps the frontend→backend boundary closed while any HIGH is open, unless the user explicitly overrides (logged).
