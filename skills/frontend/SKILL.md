---
name: frontend
description: >
  Frontend design and implementation skill for SDD projects. Reads requirements.md as its contract (not produces it). Asks the user first which of three design tracks to use: (1) External tool — Pencil.dev, (2) External tool — Figma / other, or (3) Claude Code with /impeccable pack. External tracks get a lean mood/tone brief; the Claude Code track gets a full-detail brief written inline with /impeccable loaded. External tracks: stops while user designs (backend foundation builds in the background meanwhile), then writes a frame-index handover pointing backend at the design file. Claude Code track: designs using HTML/CSS/React static mockups under specs/.../mockups/, with /impeccable enforcing craft throughout. Every track ends with handover.md for backend. Trigger on: /frontend, or when /build reaches the frontend step.
---

# /frontend — Design Brief & Handover

Input: `specs/YYYY-MM-DD-[feature]/requirements.md` — read this first. It is the contract.
Also read `product.md` in the project root — it is the product vision and full screen inventory.

If no `requirements.md` is found in a `specs/` directory: stop. "No phase spec found. Run `/ba` and `/spec` for this phase first."

## Invocation contract

| Mode | Model | Mechanism | Inputs (caller passes) | Outputs (skill produces) | Terminal state (this skill writes) |
|---|---|---|---|---|---|
| (single mode) | Sonnet | subagent (internally subagent-free — one-level nesting rule, `_shared/subagent-policy.md`) | spec directory path, phase number; reads requirements.md + product.md + mission.md | design-brief.md (always); track-dependent: design file path OR `mockups/` directory; handover.md; design-tokens.css | `frontend-complete` |

Stage 2 (design brief) is unconditional — never skipped, even if a design file already exists. Pencil MCP is forbidden during Stage 3 External-path; allowed only in Stage 5 (handover). This skill runs AS a subagent, so it never spawns its own — all brief writing is inline.

**Naming note: "Stage" vs "Phase".** The numbered Stages below (Stage 1–5) are this skill's *internal* steps for one project phase. The project Phase (Phase 1, Phase 2, … from `roadmap.md` and `requirements.md` frontmatter) is the higher-level slice the whole `/build` pipeline is working on. When you see `Phase <N>` in placeholders (wiki entry titles, "Project Phase 1" recommendation rules, "Coming in Phase N" stub label, the `Phase 1 skeleton rule` below) that refers to the project Phase. When you see `Stage <N>` it refers to the section in this skill.

## Phase 1 skeleton rule

If this is Phase 1 (check `requirements.md` frontmatter `phase: 1`): the design must cover **all screens** from `product.md` Screen Inventory, not just the screens this phase implements. Use the Phase 1 Skeleton Scope section of `product.md` to determine which screens are live vs. stubbed.

- **Live screens:** design in full — every state, every interaction
- **Stubbed screens:** design the shell (nav, header, layout, placeholder content area) with a clear "Coming in Phase N" indicator. No real content, no fake data.

The goal is to lock in the complete IA and visual language in Phase 1 so Phases 2+ fill in the skeleton rather than redesign from scratch. Stubbed screens add to the design brief's screen checklist and must appear in the handover frame index.

---

## Shared references — load before first Pencil MCP call

**Load reference: Read `${CLAUDE_PLUGIN_ROOT}/skills/_shared/pencil-preflight.md` and apply.** This is the canonical Pencil pre-flight (launch via Bash, verify with `get_editor_state`, never kill processes mid-session, escape hatch via plain-JSON Read/jq). Apply whenever this skill needs to read or write the design file — primarily Stage 5 handover, but also any later return (review surfaced a gap, backend wants a frame re-checked).

---

## Voice rules

Plain language throughout.
- Only ask about design tool choice and phase approval. Everything else is yours.
- Never "want me to / would you like" — make the call.

---

## Wiki integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/_shared/wiki.md` with `$AGENT=frontend` and `$TAGS` from `tech-stack.md`.

**Friction trigger:** the frontend compliance check (run by /build after this skill finishes) reports that screens/states from `requirements.md` are missing in `handover.md`. Title: `Phase <N> frontend friction: <area>`. Body: which screens were missed, what class of state, what to check earlier next time.

**Phase-wrap trigger:** once after the handover doc is written in Stage 4, before Completion. Title: `Phase <N> frontend: <one-line summary>` (the placeholder `<N>` is the project Phase, not the Stage). Body: which patterns the design used, what was tried and dropped, what would trip up a future frontend run.

---

## Stage 1 — Design Track Choice (asked every project phase)

**Per-phase rule:** the design track is decided **at the start of every project phase**, not locked at constitution time. Different phases have different design needs — Phase 1 establishes tokens and the screen skeleton (external tools shine here); later phases often just slot new elements into an existing system (Claude Code is faster and equally good there). Always ask. Never skip.

**Step 1 — Compute the recommendation.** Before asking, read `requirements.md` (this phase), `product.md` (full screen inventory), prior phases' `handover.md` if any, and `design-tokens.css` if it exists. Then:

- **Project Phase 1** → recommend external. The first phase locks in design tokens and the visual language for the whole product; external tools are materially stronger at this.
- **Project Phase 2+** → recommend external if **either**:
  - This phase introduces **new design tokens** — new color family, new typeface, new scale, or a new component category not yet in `design-tokens.css`.
  - This phase adds **3+ new screens** OR substantially redesigns an existing screen (not just tweaks within an existing layout).
- **Project Phase 2+** → recommend `claude-code-impeccable` if neither is true — new elements within existing tokens, on existing or near-clone screens, are a Claude Code job.

When recommending external, default to whichever external tool was used most recently (read from mission.md `## Design Tool`); fall back to Pencil if there's no prior record.

**Step 2 — State the reasoning, then ask.** Before the AskUserQuestion, write one plain-language sentence: "This phase [adds N screens / introduces token X / just adds element Y on existing screen Z], so I recommend [track]." Then ask one AskUserQuestion with **three options**, putting the recommended one first and appending "(Recommended)" to its label:

**"Design track for this phase?"**

- **External: Pencil.dev** — lean mood/tone brief; you design in Pencil; I read the finished design via Pencil MCP for the handover.
- **External: Figma / other** — same lean-brief pattern; you design in your tool; I work from screenshots/exports for the handover.
- **Claude Code (`/impeccable` pack)** — I design directly in-codebase with `/impeccable` loaded. Hierarchy, polish, motion, register-aware (brand vs product), absolute bans on common AI patterns.

**Step 3 — Record the choice.** After the user picks, write `## Design Tool` in `mission.md` with the literal value on its own line, **overwriting any previous value**:
- `external-pencil`
- `external-other`
- `claude-code-impeccable`

`/review` reads this to apply the right compliance rules for *this* phase's review (external file + frame index vs. `mockups/` + mockup index). Because the track can change per phase, mission.md must reflect the current phase's choice — not a frozen project-level decision.

**Backward compatibility:**
- Legacy `external` → treat as `external-pencil` for *this* phase's question default. The user can still choose otherwise.
- Legacy `claude-code` or `claude-code-taste` → treat as `claude-code-impeccable`, and tell the user once: "Note: the engineering-taste design track was removed — `/taste-skill` is now a code-polish pass on built mockups (see Stage 3 design quality gate)."

The answer picks which brief template + which design discipline governs Stages 2 and 3:

| Track | Brief template | Stage 2 brief writer | Stage 3 implementation |
|---|---|---|---|
| `external-pencil` | external (lean) | inline in /frontend | n/a — user designs in Pencil |
| `external-other` | external (lean) | inline in /frontend | n/a — user designs in their tool |
| `claude-code-impeccable` | internal (full) | **inline in /frontend with `/impeccable` loaded** | Claude Code designs with `/impeccable` discipline |

The two external tracks share a brief template — the difference is only which tool the user opens. On the Claude Code track, /frontend loads /impeccable once and keeps it loaded from brief writing (Stage 2) through design (Stage 3) — same lens end to end.

**`/taste-skill` is no longer a design track.** It's better used as a polish pass on already-built UI (engineering discipline applied to existing code), not as the design generator. Do not load it in this skill.

---

## Stage 2 — Design Brief

**Stage 2 is unconditional.** Skipping the design brief is a process violation regardless of: whether a design file (e.g. `companion-ui.pen`, `app.fig`) already exists, whether prior phases populated it, whether auto-mode is active, whether the dispatch prompt called any stage "polish." The brief is the contract that captures *intent for this phase* — without it the design pass and the handover have nothing to verify against. Existing design files are context, not a substitute. If you find yourself reasoning "the .pen file already exists, I can skip the brief and just append to it," stop — that is the failure mode.

**Step 0:** Run Read wiki (see Wiki integration) before writing the brief.

**Memorable-thing question:** Before writing any brief, ask one plain-text question: "What's the one thing a user should remember or feel after using this?" This becomes the north star for every design decision in the brief — write it at the top of the brief under a `## Design intent` heading.

Write `specs/YYYY-MM-DD-[feature]/design-brief.md` using the matching template. Which path runs depends on the track recorded in `mission.md` `## Design Tool`:

### Tracks 1+2 — External tool brief (lean) — `external-pencil` / `external-other`

Template: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/design-brief-external.md`

Written **inline** in /frontend (no subagent). High-level only: product context, visual style direction + reasoning, tone/mood, palette and typography intent, optional references, screen checklist.

**Do not include:** component decisions, interaction patterns, layout hierarchy, mobile breakpoints, empty/error state copy. The external tool owns those calls.

### Track 3 — Claude Code (impeccable pack) — `claude-code-impeccable`

Template: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/design-brief-internal.md`

Brief is written **inline** — this skill runs as a subagent and cannot spawn its own (one-level nesting rule). **Load `~/.claude/skills/impeccable/SKILL.md` now**, before writing a word of the brief; it stays loaded through Stage 3 so the brief and the design share one lens. Fill the schema using the impeccable vocabulary, with this discipline:

- State the register (brand vs product) up front and why — load the matching reference.
- For color: name the chosen strategy on the commitment axis (Restrained / Committed / Full palette / Drenched). Use OKLCH. Tint neutrals.
- For theme: write the scene sentence forcing dark vs light (who, where, ambient light, mood).
- For typography: hierarchy via scale + weight contrast (≥1.25 ratio).
- For layout: name the variation pattern; flag that "cards are the lazy answer."
- For every section: explicitly call out which absolute-bans you are NOT triggering.
- For motion: ease-out exponential curves only.
- Write the memorable-thing answer at the top under `## Design intent` — it is the north star for every decision below it.

Note: /impeccable's PRODUCT.md / DESIGN.md gates can be SKIPPED for brief writing — those gates are for /impeccable craft / shape commands. You are filling the design brief schema, not invoking impeccable's own pipeline. Apply the design laws and register-specific reference; skip the setup gates.

Before continuing, self-verify `design-brief.md` contains the register + color strategy + scene sentence + screen checklist — every section of the schema filled.

---

## Stage 3 — Design

### Tracks 1+2 — External tool (`external-pencil`, `external-other`)

**Hard rule — Pencil MCP is forbidden during Stage 3 External.** Calling any `mcp__pencil__*` tool (or the Figma / other-tool equivalent) during Stage 3 External-path is a process violation. The user drives the design pass in their tool — Claude's only job in this stage is writing the brief (Stage 2) and waiting. Claude returns to Pencil MCP in **Stage 5** to read the finished design for the handover. If auto-mode is active, that does NOT authorize designing in Pencil on the user's behalf — auto-mode minimises *interruptions*, not *user authorship of the design*. The screen checklist hand-off below is a hard stop, not a soft prompt.

Tell the user:

> **Design brief written.** Open your design tool, share the contents of `specs/[date]-[feature]/design-brief.md` as context, and cover every screen on the checklist below.
>
> **Screen checklist ([N] states to cover):**
> [paste the numbered checklist from the brief]
>
> Come back here when the design is done and you're happy with it. Meanwhile I'll build the data layer and behind-the-scenes plumbing that doesn't depend on the design — it'll be done by the time you're back.

Stop. Do not proceed until the user returns. (The foundation build itself is orchestrated by `/build` Step 4b, not this skill — this skill just sets the expectation in the stop message.)

When the user returns: go to Stage 4.

### Track 3 — Claude Code designs (`claude-code-impeccable`)

**Load /impeccable first.** Before any mockup work, load `~/.claude/skills/impeccable/SKILL.md`. Apply its register-aware reference (brand vs product), color strategy, theme scene-sentence, and absolute bans to every screen. (You can skip impeccable's PRODUCT.md / DESIGN.md gates here — those are for /impeccable's own pipeline; you're using the design laws and register reference, not invoking impeccable's craft/shape commands.)

**Do not load /taste-skill as a design generator** — it is no longer a design track. (It runs as a code-polish pass after mockups exist; see the design quality gate below.) If the design brief was written under the legacy `claude-code-taste` value, treat it as `claude-code-impeccable` per the backward-compat rule above.

Design every screen from the checklist using the project's actual tech stack (from `tech-stack.md`). Produce static mockup files — real components with hardcoded data, no API calls, no routing logic. The goal is a pixel-accurate preview of every screen and state, not a working app.

**How to design:**
1. Create `specs/YYYY-MM-DD-[feature]/mockups/` directory.
2. For each screen group (home, board, dialogs, etc.), write a self-contained HTML file using the project's stack (Tailwind, shadcn tokens if applicable, or plain CSS matching the design brief's style decisions). Each file renders all states of that screen group side by side. Apply the loaded skill pack's discipline throughout — this is the whole point of the track.
3. Start a local preview: `open` the HTML files in a browser, or run a minimal static server if the stack requires it.
4. Use the `browse` skill (or browser screenshots via MCP) to capture each file. **Self-review before user-review (non-optional):** after the first capture of a screen group, look at the screenshot critically and list 3 specific issues — hierarchy slips, spacing drift, AI-tells from /impeccable's bans, register confusion, trunk-test failures, anything that looks generic. Fix them. Re-screenshot. Only then show the user every 4–5 screens with a brief description. Sonnet is much sharper as a critic of an existing image than as a generator on the first pass — this self-critique loop is where the quality gain lives.
5. One `AskUserQuestion` mid-way: "Direction right?" (options: "Keep going," "Adjust this," "Change the style"). Adjust once if needed, then continue.
6. Complete all screens from the checklist before going to Stage 4.

**Design quality gate (Claude Code tracks only):** Before Stage 4, run these checks on every screen. All three are blockers.

**Trunk test** — on each screen, can you instantly answer all four questions without thinking?
1. What product/site is this?
2. What page am I on?
3. What are the major sections?
4. What can I do from here?

A FAIL on any screen means the information hierarchy is broken. Fix before approval.

**AI slop check** — /impeccable's bans are authoritative. Use the "Absolute bans" + "AI slop test" + "Category-reflex check" from `~/.claude/skills/impeccable/SKILL.md` — side-stripe borders, gradient text, glassmorphism-as-default, hero-metric template, identical card grids, modal-as-first-thought, em dashes in copy, category-reflex palettes (observability→dark blue, healthcare→white+teal, etc).

In addition, baseline bans: purple/violet gradient backgrounds, decorative blobs / wavy SVG dividers, uniform bubbly border-radius on everything, centered layout on every element.

**Code-polish pass (`/taste-skill`)** — load `~/.claude/skills/taste-skill/SKILL.md` and run it across every mockup file. /impeccable governs design language; /taste-skill governs *implementation craft* — the layer below the design language. Apply its rules across the mockups: anti-`Inter` / anti-purple-glow, dependency verification before any new import, `min-h-[100dvh]` instead of `h-screen`, CSS Grid instead of flex-percentage math, no 3-equal-card feature rows, no generic names ("John Doe") or round-number fake stats ("99.99%"), no Unsplash links, hardware-accelerated transforms only. This is the last filter before user review — the pass that turns "Sonnet output" into "premium Sonnet output." Only load /taste-skill *here*, after mockups exist; never during the brief or initial generation (it would conflict with /impeccable's design-language ownership).

Do not use Pencil MCP tools during Stage 3. Do not call Pencil MCP from Claude Code tracks at all (Pencil MCP is for `external-pencil` Stage 5 only).

---

## Stage 4 — Approval & Spec Gap Check

Ask one `AskUserQuestion`: "Design done — anything to flag before the handover doc?" (options: "Looks good, proceed," "One thing to flag," "Need to revisit the spec")

If a spec gap surfaces (e.g., the design revealed a needed API not in requirements.md): stop. "Design requires [endpoint] which is not in requirements.md. Update the spec before backend starts." This is a spec issue — do not work around it.

If approved: proceed to Stage 5.

---

## Stage 5 — Handover Doc + Design Tokens

**STOP — run the pre-flight at the top of this skill before any MCP call.** If Pencil isn't open on WSL2, every step in this stage fails. Do NOT prompt the user to open Pencil — launch and **focus** the `.pen` headlessly per the pre-flight's Step 1 + Step 1b (single second-instance launch with the file path, `sleep 20`, verify the active editor with `get_editor_state`). The `filePath` arg to MCP reads is ignored — the target file must be the *focused* tab, so Step 1b is mandatory, not optional. If headless focus won't take after a couple of single-launch-then-wait tries, fall back to reading the `.pen` as plain JSON (escape hatch) rather than blocking on the user.

This stage produces **two artifacts**:

1. `specs/YYYY-MM-DD-[feature]/handover.md` — the frame index (per the schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/handover.md`)
2. `specs/YYYY-MM-DD-[feature]/design-tokens.css` — generated token file the backend imports directly

### Step 1 — Generate `design-tokens.css`

Pull the design system variables from the design file (not from the brief, not from memory):

- **Pencil:** call `mcp__pencil__get_variables({ filePath })`. The result is a variable map with typed values (color / number / string). Emit as CSS custom properties under `:root`.
- **Figma / other:** use whatever token export the tool offers.

The file backend imports. Keep it mechanical — one variable per token, no editorial reshaping:

```css
/* Generated from [design file path] on [date]
 * Source: mcp__pencil__get_variables
 * Do not edit by hand — regenerate if the design file changes.
 */
:root {
  --surface-primary: #F4F2EF;
  --border-subtle: #EEECE8;
  --accent-primary: #C8B496;
  --fg-primary: #1A1A1A;
  --fg-secondary: #4A4A4A;
  /* ... every color token ... */
  --font-heading: "Newsreader", serif;
  --font-body: "Inter", sans-serif;
  --font-mono: "Geist Mono", monospace;
  --font-size-heading: 15px;
  --font-size-body: 13px;
  /* ... every size token ... */
}
```

If the design file uses fonts that aren't yet loaded in the app (common first-phase case), include a **Fonts required** section in the handover so backend knows to wire `next/font/google` or equivalent.

### Step 2 — Write `handover.md`

**The handover is an index, not a specification.** The design file is the spec. Backend will open the design file and build from it — your job here is to make that navigable, not to narrate every visual detail.

Include (per schema):
- **Design file — source of truth:** absolute path, tool type, and explicit "how to read it" instructions. Mark loud and clear: *"Backend must open the design file and build from it. Visual details are in the file, not in this doc."*
- **Design tokens:** point to `design-tokens.css` generated in Step 1. Describe the import path (e.g. `@import './design-tokens.css'` from `globals.css`).
- **Fonts required:** list families + weights. Backend wires font loading.
- **Frame index:** for every state listed in `requirements.md` UI Requirements, give the exact design-file frame/node the backend should read. Every state maps to a frame, or is marked "not designed" with a reason.
- **Reusable components:** design symbols → React component names backend should create.
- **Deviations from requirements:** anything designed differently than the spec said. If none, state explicitly.
- **Layout / IA notes:** only flag structural patterns the implementer might miss by pattern-matching against existing code (e.g. "no sidebar — top header pattern throughout"). Do not narrate visuals.
- **API contracts expected from backend:** copied directly from requirements.md API Contracts section.

What NOT to include:
- Do not paraphrase visuals ("3-column project grid, status chip per card"). The design file shows this; your description adds no information and risks drifting from what's designed.
- Do not list colors, sizes, spacing values. Those come from `design-tokens.css`.
- Do not add a state-by-state narrative. A frame index row is enough.

---

## Completion

**Pre-Completion:** Run the phase-wrap Write learning (see Wiki integration) — one entry per phase.

**Checkpoint state file (compaction-safe handoff):** before output, write `.build-state.json` with `step: "frontend-complete"`. Preserve every other field (`phase`, `feature`, `reviewIteration`, `requirementsHash`, `dogfoodPid`); null `currentSubStep`. This is the resume anchor — if context is compacted between this skill ending and `/backend` starting, the next `/build` reads `frontend-complete` and jumps straight to Step 3 (Backend) instead of replaying design discovery and handover writing.

> **Frontend complete.**
> **Screens:** [count] designed across [count] states
> **Design track:** one of `external-pencil` / `external-other` / `claude-code-impeccable` — name the actual choice and where the design lives (e.g. "claude-code-impeccable — mockups in specs/<dir>/mockups/" or "external-pencil — design file at app.pen")
> **Design brief:** `specs/YYYY-MM-DD-[feature]/design-brief.md`
> **Handover:** `specs/YYYY-MM-DD-[feature]/handover.md`
> **Ready for:** frontend compliance check → backend

---

## Ground rules

- requirements.md is the contract. The brief translates it — nothing more, nothing less.
- Design brief is always written before any design work starts, regardless of path.
- During design (Stage 3 — External tool path): Claude does not call Pencil MCP tools. The user drives the design in their tool.
- During handover (Stage 5) and after: Claude MAY call Pencil MCP tools (or the design-tool MCP equivalent) to read the finished design — e.g. to confirm frame IDs, extract reusable component names, or verify coverage against the checklist. Backend is expected to read the file via MCP during implementation.
- **Pencil MCP requires the desktop app to be open on WSL2.** Run the pre-flight at the top of this skill before any `mcp__pencil__*` call. Never speculatively retry on `WebSocket not connected` — that just wastes turns; prompt the user to open the app instead.
- If design reveals a spec gap, stop and surface it — never silently work around a missing API.
- No approved design = skill incomplete. Do not write the handover without explicit approval.
- The checklist at the end of the brief is the coverage contract — every item must be accounted for in the handover.
