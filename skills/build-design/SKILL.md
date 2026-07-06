---
name: build-design
description: >
  The design phase of the /build SDD stack. Picks one of two tracks and applies a single UX / craft / accessibility gate catalogue to the phase's UI. **claude-code** — Claude designs the interface itself in-codebase as static HTML/CSS/React mockups the user sees rendered at every milestone and fork, gated on a must-pass gate report; the mockups ARE the design, so there is no handover document — backend builds from them directly and from any non-visual decisions logged to docs/decisions.md. **external** — Claude writes a tool-agnostic design brief, hands it to the user, and waits; when the user returns with their exported design images, Claude reviews them against the same gates, writes design-comment.md (the change list), and leaves a bare screen→image index so backend knows which picture is which screen. Reads requirements.md; writes design-brief.md + design-tokens.css (+ mockups/ + gate report on claude-code; + design-comment.md + a screen→image index on external). Runs inline. Invoked by /build at the design step, or standalone via /build-design.
---

# build-design — pick a track, design or review, gate, hand off to backend

This skill owns one thing: the **UX / craft / accessibility bar** for the phase's UI, expressed as a single gate catalogue (`references/gates.md`). It applies that bar on whichever of two tracks the phase uses:

- **`claude-code`** — Claude designs the interface itself, no external tool, as static HTML/CSS/React mockups the user can see and click before anything is wired. Kept structurally correct, accessible, on-craft, and **gated on a must-pass gate report** — with the user looking at rendered pixels at every milestone and fork. The mockups live in the repo and **are** the design; backend builds from them, so this track writes **no handover document**.
- **`external`** — the user drives the design in whatever tool they like. This skill writes the brief, waits, then — when the user returns with exported images — **reviews the design against the same gates**, writes `design-comment.md` (the precise change list), and builds a bare **screen→image index** so backend can find each screen. No tool-specific mechanics: the user provides exported images, and that is all this track consumes.

The **app-shell spec** (`references/app-shell-spec.md`) is not a runtime mode of this skill — it's a static reference that **build-spec** reads to fill `requirements.md`'s App-Shell section. It lives here because the shell it specifies is exactly what this skill designs and gates; this skill never invokes it.

> **Judgment, not lookup.** Trust your design read. The instructions fix *contracts* — what must be true, what the gate report must contain. How you phrase things to the user is yours; a quoted string is intent to convey, not a script to recite. User-facing text is plain language, never file paths or stack names (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Field | Value |
|---|---|
| **Model** | Opus (the design brief is Opus-authored). |
| **Mechanism** | **Inline.** This skill holds live user gates AND shows the user rendered PNGs at every fork — a subagent can do neither (`_shared/subagent-policy.md` Rule 1). It fans out **Sonnet leaf workers** for the things that don't decide anything: render HTML→PNG, self-critique vision, read the user's exported design images. |
| **Inputs** | `track` ∈ `claude-code` \| `external` (arg; absent → resolve as a fork). `requirements.md` (the contract). `product.md` (screen inventory + Design North Star). `outcome-card.md` (reference only — the user-approved outcome, plain language). `design-tokens.css` if a prior phase set it. |
| **Outputs** | `design-brief.md` (always). `design-tokens.css` (always). **`claude-code`:** `specs/<phase>/mockups/` + a gate report; non-visual decisions logged to `docs/decisions.md`; **no handover doc.** **`external`:** the user's exported images + a bare screen→image index (`handover.md`) + `design-comment.md`. |
| **Terminal step** | **NONE.** Return without writing `.build-state.json`. The orchestrator runs the **design-compliance** gate: pass → it writes `design-complete`; fail → it rolls `step` back to `spec-complete` and re-runs this skill. |

`requirements.md` is the contract — the brief translates it, nothing more. No `requirements.md` in a `specs/` dir → stop: "No phase spec found — spec must run first."

**Standalone use** (`/build-design` typed directly, no active build): skip the build-stack file contracts. Treat the user's request as the brief — `claude-code` → design + gate it against the same catalogue; `external` → point it at the exported images to review and write the change list. Pick a working directory, run the same flow, obey the same one rule. No orchestrator gate follows, so surface the gate report / verdict to the user yourself and stop.

## The one rule (both tracks obey their half)

**Never settle a UI decision the user will see or feel with words alone.** On `claude-code` that means: every milestone and every fork, the user sees a *rendered mockup*, not a description of one — prose proposes, pixels decide. On `external` the equivalent is *evidence per finding* — every `design-comment.md` line names the screen and the exact change, never a vague "improve the spacing." This is a hard requirement; the **visual-shown log** (claude-code) and the per-finding gate IDs (external) are the proof it was honored.

---

## Track resolution (first action)

Track is an explicit arg. Absent → it's a **felt tooling fork** — resolve it, don't guess. Check `docs/decisions.md ## User decisions` first (a settled track is honored, not re-asked). Otherwise recommend by the principle *"does this phase establish new visual language?"*, then one `AskUserQuestion`, recommended option first:

- **Phase 0 (Foundation)** → recommend **external** — it designs every screen and locks the tokens + visual language; a dedicated design tool is materially stronger at the whole-product visual pass.
- **Phase 1+** → **external** if the phase adds a *new visual vocabulary* (new token family, a component category not in `design-tokens.css`, ~3+ new screens, or a `rebuild`); **`claude-code`** otherwise (new elements within existing tokens, on existing/near-clone screens).

Record the chosen track (`claude-code` / `external`) to `mission.md ## Design Tool` (`build-review` reads it back to pick compliance rules) and the fork to `docs/decisions.md`.

**North star — once, then carry forward.** `mission.md ## Design North Star` absent → ask one plain question ("what should a user remember or feel after using this?"), write it there, copy it to the brief's `## Design intent`. Present → read silently, copy it. One app, one north star.

---

## Track: `claude-code` — design it, gate it, no handover

### Step 1 — internal design brief (unconditional)

Write `specs/<phase>/design-brief.md` from `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/design-brief-internal.md`. **Load `references/craft.md` now** and fill the schema with its discipline: **register** (brand vs product) up front; **color strategy** on the commitment axis (Restrained/Committed/Full/Drenched, OKLCH, tinted neutrals — never the timid default); a **theme scene-sentence** forcing dark vs light; **type hierarchy** by scale + weight (≥1.25); a named **layout pattern** that isn't the reflex ("three equal cards" is the lazy answer); each section calling out which anti-slop ban it is NOT triggering; motion = ease-out; north star at top under `## Design intent`. Skipping the brief because a design exists is the process violation — never skip it. Self-verify before continuing: register + color strategy + scene sentence + screen checklist all present.

**Phase 0 rule.** `requirements.md` frontmatter `phase: 0` → design **every screen in `product.md`'s Screen Inventory** (the whole product, not one feature) to final visual polish: static, realistic mock data (not lorem, not "Coming soon"), pixel-complete, clickable, unwired. This locks the visual language for the whole product; Phases 1+ inherit it (except a `rebuild`). Every screen appears in the brief's checklist.

### Step 2 — the four milestones (ladder, not recital)

Design under `specs/<phase>/mockups/` — one self-contained HTML **per screen group** (the *authoring* unit; all groups are assembled into one review page at M3 — "per screen group" never means "one group per turn"), all states side by side, real components + hardcoded data, no API/routing, in the project's actual stack (`tech-stack.md`). Keep a shared token layer (a `:root`/theme file the mockups import) — that layer becomes `design-tokens.css`. Structural + craft rules apply throughout: `references/ux-rules.md` (navigation, no-dead-ends, forms, a11y) and `references/craft.md` (the taste layer). Run the four milestones in order; **show the user a rendered view at each** (see *Image hygiene* below) and **self-critique before every show** (render → name 3 concrete issues → fix → re-shoot; the user sees the second draft, never the first):

**One continuous pass — the milestones are shows, not stops.** Design the phase's *entire* UI in a single turn. M1 (map) and M3 (mockups) are **shows**: render it, put it in front of the user, and **keep going** — never end the turn after a screen group, never say "I'll do the next group next." The **only** two places this track pauses for the user are each **M2 direction fork** and the **Step-4 approval**; everything between them is one uninterrupted build. The orchestrator's auto-continue rule applies *inside* this skill too — a milestone is not a gate. The whole phase is designed and assembled into one review before the approval.

| # | Milestone | Artifact shown | The check it clears |
|---|---|---|---|
| **M1** | **Structure map** — before any pixels | the wireflow (Mermaid graph: nodes = screens, edges = navigation, each node annotated with its states) | the three structural reads below — fix until `0 orphans, 0 dead-ends, all states present` |
| **M2** | **Look anchor** — one screen (the primary-outcome screen) to final polish | the anchor, usually **as a fork** (2 directions you're torn between — color commitment, layout density) | the visual language is decided; the approved anchor locks tokens + language for every other screen |
| **M3** | **Full mockups** — every checklist screen, every state | **one self-contained interactive review artifact** for the whole phase — every screen + every state on a single published page (assembling the per-group mockups), not a trickle of per-group PNGs | the coverage checklist is the completeness contract — an unbuilt screen is not done |
| **M4** | **Gate report** — the Definition of Done | the review artifact alongside the report | every applicable gate green-or-justified (below) |

The three **structural reads** at M1 — run over the *map*, because routes don't exist as code yet; they are gates `G-REACH` / `G-STATES` / `G-NAV` checked here:

- **Reachability** — every screen has ≥1 inbound path from a start point. No island.
- **Exitability** — every screen has a forward action AND a back/close path. No dead-end. (404 → home; empty state → the action that fills it; confirmation → next step.)
- **States drawn** — every data screen lists Loading · Empty · Error · Not-found · Offline, not just the happy path.

### Step 3 — the gate report (Milestone 4)

You may not report the design complete while any **applicable** gate is unknown or failing. Run every applicable gate from `references/gates.md` over the finished mockups (they are real HTML, so axe / token / visual gates run *directly* against them; the structural gates run against the M1 wireflow). Gates scope themselves ("applicable when") — a marketing page isn't blocked by the auth gate. Emit the report **verbatim in this shape**:

```
## Gate report — Phase <N> design
| Gate        | Tier | Applies | Status | Evidence |
|-------------|------|---------|--------|----------|
| G-REACH     | 1    | yes     | PASS   | wireflow: 0 orphans, 0 dead-ends |
| G-STATES    | 1    | yes     | PASS   | state matrix: all 5 per data screen |
| G-A11Y-AXE  | 1    | yes     | PASS   | axe: 0 violations (see .review/axe.json) |
| G-A11Y-TARGET | 1  | yes     | PASS   | 0 nodes < 44px |
| G-TOKENS    | 1    | yes     | PASS   | 0 raw hex/px outside token layer |
| G-DENSITY   | 2    | yes     | PASS   | .review/<screen>.png · all balanced |
| G-DEFECTS   | 2    | yes     | PASS   | .review/<screen>.png · 0 defects |
| G-AUTH-SET  | 1    | n/a     | —      | no accounts this phase |
| ...         |      |         |        |          |

### Visual-shown log
- M1 structure map · wireflow · shown ✓
- M2 look anchor · dashboard · interactive artifact · shown ✓
- fork · table vs cards · side-by-side artifact · shown ✓
- M3 phase review · all screens + states · interactive artifact · shown ✓
- M4 gate report · alongside the review artifact · shown ✓
```

Any Tier-1 gate FAIL blocks on its output; any Tier-2 gate blocks on a missing artifact. A non-applicable gate is `—` with the reason — never silently dropped (a dropped gate reads as "covered" when it wasn't). Fix and re-run until green-or-justified. The visual-shown log must have a line for every milestone and fork — an empty log fails the Definition of Done.

### Step 4 — iterate on the artifact, approve, then close out

The user reviews the M3 review artifact. Iterate on it in place — revise the mockups, re-publish the **same** artifact (same URL), repeat until they're happy — then one `AskUserQuestion`: design done — anything to flag before backend? (proceed / one thing to flag / need to revisit the spec). Spec gap surfaces (the design needs an endpoint or screen not in `requirements.md`) → **stop and name it** ("design needs X, not in the spec — the spec must update before backend"); never invent past it. Approved → close out (below) and return. **No handover document** — the mockups are the design and they're in the repo.

**Close-out (claude-code):**
1. **`design-tokens.css`** — write the mockups' shared token layer to `specs/<phase>/design-tokens.css` (one CSS custom property per token under `:root`, a generated-from header). Backend imports this one file so tokens aren't re-extracted per component. Fonts not yet in the app → note them in the decisions log so backend wires them.
2. **`docs/decisions.md`** — log every **non-visual** decision the mockups don't show on their face: a structural choice and its why (slide-over not modal, wizard not one form), any deliberate deviation from `requirements.md`, component→name mappings an implementer might miss. This is the *only* design knowledge a cold backend session can't read off the mockups — so it must land in the living doc that survives a new session or compaction. What IS visible in the pixels (colors, spacing, layout) is never restated here.

Backend reorients from: the mockups in `specs/<phase>/mockups/` (the design), `requirements.md` (the screen list to map them against), `design-tokens.css`, and `docs/decisions.md`. That set is complete with no handover doc.

---

## Track: `external` — brief, wait, review, gate, index

### Step 1 — external design brief (unconditional)

Write `specs/<phase>/design-brief.md` from `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/design-brief-external.md`. **Include:** product context, product type & mental model, full spec (screens, states, behaviors, forward-compat callouts, IA, user stories), real on-screen copy, the numbered screen checklist. **Exclude — the design tool owns the entire look AND layout:** any styling (visual direction, palette, typography, mood, theme), component decisions, layout hierarchy, interaction patterns, breakpoints, spacing. Carry the north star into `## Design intent`.

### Step 2 — hand off and wait

Convey: the brief is written; design it in whatever tool you like, cover every screen on the checklist (paste the numbered list), and **export the screens as images when you're happy.** Then **stop** — do not proceed until the user returns with their images. There is **no background build** and nothing runs in the meantime — the design is the only thing in flight. Do not design on the user's behalf.

### Step 3 — review the exported design against the gates

The user returns with exported images → review them against the **same `references/gates.md`**, adapted to a non-runnable design (the split shifts toward evidence/vision; the catalogue is identical). First map every screen in `requirements.md` (the coverage contract) to its exported image — this mapping *becomes* the screen→image index. Then judge each image:

- **Structural — from the index + requirements** (deterministic): every required screen present? every data screen's five states (Loading/Empty/Error/Not-found/Offline) designed? every non-home screen has a back affordance? any orphan or dead-end? auth set complete where accounts exist? (`G-REACH` `G-STATES` `G-NAV` `G-AUTH-SET` `G-SCREEN-JOB`.)
- **Visual — vision pass per image** (`G-DENSITY`, `G-DEFECTS`, `G-A11Y-TARGET`, contrast): give each image a density verdict (too busy / balanced / too sparse) and a concrete-defect list (overlap/collision · clipping/cut-off · broken image · empty control · >1 equal-weight primary CTA); flag *likely* contrast failures and undersized targets. Do **not** score holistic hierarchy or taste — that stays the user's call on their own design. Be honest — contrast off an image is an estimate; flag the pair and tell the designer to verify with a checker, don't assert a false-precise ratio.
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
2. **`handover.md`** — a **bare screen→image index only**, per `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/handover.md`. Backend can't read an image the way it reads a mockup, so it needs the map: every screen in `requirements.md` → its exact exported image file (or "not designed" + reason), plus the design source path (marked loud "backend builds from these images"), the tokens import path, fonts required, and any structural note an implementer might miss. **Not a narration** — no restating colors/sizes/spacing (they're in the images + tokens); just the map.

---

## Image hygiene — how the design reaches the user, and how the agent checks its own work (mechanism inline)

Two separate paths, because they have different constraints.

**User-facing shows → a self-contained interactive Artifact.** The **M2 fork comparison** and the **M3 phase review** reach the user as a published **Artifact**, never a flat screenshot (this user reads pixels far better in an interactive page than in a PNG). The mockups are already self-contained HTML, so assemble the phase's screens — every group, every state — into **one page**: a thin gallery wrapper, one labelled section per screen (load the `artifact-design` skill for the wrapper's polish; the mockups carry their own design), and call the **Artifact tool** with that file. A fork is the same page built as a side-by-side comparison of the 2–3 options. **Iterating = edit the assembled page and re-publish to the *same* URL** — one artifact per phase, revised in place, not a new link each round. Surface the artifact URL to the user; add the line to the visual-shown log.

**Self-critique → a render leaf that returns text only (`_shared/subagent-policy.md` Rule 8).** The agent's own before-every-show QA never puts image bytes in the main session. Spawn one **Sonnet render leaf** with a concrete brief: serve/open the target HTML, screenshot it with **`/browse`** (gstack, session-isolated) to `specs/<phase>/mockups/.review/<name>.png` (shoot 390/768/1280 for `G-RESPONSIVE`; inject axe-core and save `.review/axe.json` when the brief asks). The leaf **returns the saved PATH + a short text verdict** (density verdict + defect list, axe violation count) — **never the image bytes.** The skill decides on the text; the `.review/*.png` are the gate evidence behind the report, not something the main session holds. This is the self-critique loop from Step 2 (render → name 3 issues → fix → re-shoot) and any critic pass — a Sonnet leaf renders and checks, the skill decides.

Brief a concrete task + file list, never "read and run a SKILL.md." The render leaf's brief is literally:

```
Render <abs path to .html> with /browse. Screenshot to <abs .review/<name>.png>
at 390/768/1280px. [Inject axe-core and save <abs .review/axe.json>.] Return:
the saved PNG path(s), and a one-line verdict — a density verdict (too busy /
balanced / too sparse) + a concrete-defect list (overlap · clipping/cut-off ·
broken image · empty control · >1 equal-weight primary CTA, or "0 defects")
[+ axe violation count]. Do NOT return image bytes.
No spawning agents, no /build*, no claude -p, no commit, no server left
running, don't address the user — return content/paths only.
```

**Every leaf brief ends with that containment line** — it is what keeps a leaf from spawning a second level, committing, or talking to the user in this skill's name. (`external` reviewing exported images uses the same hygiene: a leaf reads the image files and returns text findings + paths; the main session triages on text.)

---

## Forks — the felt decisions the user picks (`_shared/voice.md`)

A **fork** is any UI decision the user will *feel* and that is expensive to reverse: structural (table vs. cards, wizard vs. one long form, modal vs. dedicated page, inline-edit vs. detail page, top-nav vs. sidebar) or visual (color commitment, layout density, hero treatment). Fine styling is *not* a fork — that's the design's job, approved at the gate. When you hit a fork, on `claude-code` you do **not** ask in prose:

1. **Build both** — a small, real mockup of each option (2, occasionally 3), same content, so the difference *is* the decision.
2. **Assemble them side by side** into one comparison page and **publish it as an Artifact** (Image hygiene above); self-critique each option with a render leaf first (text-only). Put the interactive comparison in front of the user.
3. **Then** `AskUserQuestion` — recommended option first and labeled `(Recommended)`, each option one plain sentence of the tradeoff, the question referencing the published comparison. A real recommendation, not a neutral menu.
4. **Log** the fork in the visual-shown log and, once picked, in `docs/decisions.md ## User decisions`.

Fork the direction-setting calls (a handful per phase); default the rest and show them at the milestone. **Check `docs/decisions.md` before surfacing any fork** — a settled decision is honored, never re-litigated or quietly reversed. On `external` the felt forks live in the brief's spec choices; the same ledger discipline applies.

---

## References & schemas (load as needed)

- `references/ux-rules.md` — the structural UX ruleset (navigation, no-dead-ends, flows/IA, the complete auth set, forms, a11y floors, the codeable token/spacing/type system). Read at M1, keep open. Source of most gates.
- `references/craft.md` — the taste layer (register, committed color, type hierarchy, layout variation, anti-slop bans, trunk test, motion). Read at M2 / at brief-writing time.
- `references/gates.md` — the gate catalogue, shared by both tracks: each gate's tier, what it verifies, how to check it with existing tooling, the evidence it must produce. `claude-code` reads it at M4 (skim at M1 to build toward it); `external` runs the whole catalogue over the exported images.
- `references/toolkit.md` — the vision loop, running axe on static mockups, free asset recipes (hand-SVG, Mermaid, Lucide/Iconify, @vercel/og, matplotlib/QuickChart), optional env-key raster generation, per-platform component defaults.
- `references/app-shell-spec.md` — the standard app-shell spec **build-spec** reads to fill `requirements.md`'s App-Shell section; also a useful shell reference for M1 when the phase builds foundation chrome. This skill does not invoke it.
- Schemas: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/{design-brief-internal,design-brief-external,handover}.md` (`handover.md` = the external track's screen→image index only). Voice: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`. Wiki learnings: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/wiki.md`, `$AGENT=design`.

---

## Ground rules

- `requirements.md` is the contract; the design serves it. A design that needs something not in the spec → **stop** and surface the gap; never invent past it.
- The brief is always written before any design work — even if a design already exists.
- The coverage checklist (brief / M1) is the completeness contract — every screen reaches the mockups (claude-code) or the screen→image index (external) and the gate report.
- **Show before decide** (`claude-code`): no milestone without a shown mockup, no fork without a shown comparison; the visual-shown log proves it. **Evidence per finding** (`external`): every `design-comment.md` line names the screen, the gate, and the exact change.
- Evidence beats assertion — a gate is green only with an artifact behind it. Self-critique produces the second draft the user sees, never the first.
- Rendering is `/browse` only (gstack, session-isolated) — no other browser/test framework (`references/toolkit.md`).
- **No handover doc on `claude-code`** — the mockups are the design; backend reads them + `requirements.md` + `docs/decisions.md`. `external` writes only the bare screen→image index because its design isn't code backend can read.
- No terminal `step` — return; the orchestrator gates design-compliance and writes `design-complete` (or rolls back to `spec-complete` and re-runs this skill).
- Not approved / a HIGH still open = not done. `claude-code` closes out only after an all-green-or-justified gate report + explicit approval; `external` keeps the frontend→backend boundary closed while any HIGH is open, unless the user explicitly overrides (logged).
