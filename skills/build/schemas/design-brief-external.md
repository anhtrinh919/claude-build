# Design Brief — External Tool Path

Brief for handoff to whatever external design tool the user chooses. The external tool owns the **entire visual look** — style, color, typography, layout, components, spacing, and motion. This brief provides **context, spec, and copy only**: deep grounded product context, the full spec (screens, states, behaviors, IA), and the real on-screen copy — so the tool doesn't design Phase N into a corner that later phases must retrofit around, and so it knows the *why* behind every screen group, not just the *what*.

**Length target:** 15–40K tokens. Lower bound for small phases, upper bound for foundation phases with rich product context. Stay under 50K to leave the design tool's context budget room for its own design system, screen iteration, and tool overhead.

**Read before writing:** `mission.md`, `product.md`, `tech-stack.md`, `roadmap.md`, `specs/YYYY-MM-DD-[feature]/requirements.md`, and any project-level `CLAUDE.md` or `past-lives.md`. Pull from them — do not paraphrase from memory.

**Do NOT include:** visual style direction, color/palette intent, typography intent, tone/mood-as-aesthetic, aesthetic references, theme posture (light/dark), specific component names, layout hierarchy, interaction patterns, mobile breakpoints, padding/spacing values. The external design tool decides the entire look and layout; prescribing it defeats the point of an external track and biases the tool. (You CAN describe behavioral *intent* — "the empty state should feel confident, not hand-holding.")

**DO include** (the lessons from past phases — every section below is required unless marked optional):
- Full product vision — what the product becomes at the end of the roadmap, not just this phase.
- A roadmap-at-a-glance.
- Product type & mental model — concrete product type and which design approach/toolset to apply (and which to avoid).
- Forward-compatibility callouts — "leave room for X, which lands in Phase Y." This is the single most important section for preventing retrofit pain.
- Per-screen-group rationale — what each group of screens does and why it exists.
- User stories — verbatim from `requirements.md`, organized by screen group.
- Patterns to avoid — product-category and mental-model anti-patterns (not visual bans).
- Copy & content — real on-screen text, labels, and microcopy (or pointers to the canonical content source).
- Information architecture table.
- Screen checklist with coverage notes.

---

## Design intent

One short paragraph — the answer to the "what's the one thing a user should remember or feel after using this?" memorable-thing question. This captures the **experience and feeling goal**, not a visual instruction. Quote the user verbatim where possible. End with a one-liner: *"Every design decision should be weighed against this."*

## Product type & mental model

State two things explicitly:

1. **Concrete product type (one line).** Name what this is: "a production native tablet/iPad app," "a CLI tool," "a data dashboard web app," "a marketing site," "a B2B SaaS web app," etc. The more specific, the better — this tells the design tool which platform conventions and native patterns to draw from.

2. **Which mental model and toolset to apply — and which to avoid.** AI design tools default to building an interactive HTML artifact (scrollytelling page, slide deck, visual playground) unless told otherwise. Be explicit. Example for a native app: *"Design this as a real native application using standard platform app UI patterns (navigation, screens, touch targets, system chrome). Do NOT design it as an interactive HTML document, a scrollytelling or explorable web page, a slide deck, or a code artifact."* Adapt to the actual product type — for a marketing site, the HTML-document mental model is correct; for a mobile or tablet app, it is wrong.

**Why this matters:** without this section, AI design tools apply the wrong mental model by default — usually an interactive webpage — regardless of the product type described elsewhere in the brief. Name it explicitly so the tool reaches for the right conventions from the start.

## Product context — full vision

The full picture of what the product becomes at the end of the roadmap. Aim for **3–6 paragraphs**, not just a summary. Cover:
- **What it is.** Product type, deployment surface, who runs it, who uses it.
- **What it replaces.** The current tools or workflows the user is moving away from. Naming these sharpens the contrast for the design tool.
- **What it explicitly isn't.** Adjacent products that are NOT the target — this prevents the design tool from pattern-matching toward the wrong reference.
- **The end-state form factor.** Describe the eventual layout, navigation model, and key surfaces in prose. the design tool designs Phase 1 better when it can picture the Phase N destination.
- **The operator's day with this product.** Optional but high-value — a short narrative of the user's typical day using the product. Helps the design tool see screens as connected nodes in a flow, not isolated mockups.

Pull from `mission.md` and `roadmap.md`. Don't paraphrase — quote where useful.

## Patterns to avoid

This section is about **product-category and mental-model anti-patterns** — wrong-category thinking the design must not fall into. It is NOT a list of visual bans (the design tool picks the visuals).

If the project has predecessors (failed attempts, cousin projects, prior prototypes), name the product-level failure modes. Read `past-lives.md`, `CLAUDE.md`, or any "lessons learned" docs.

For each anti-pattern:
- **Name it.** What product category or mental model is it being confused with?
- **What went wrong.** What did the design become that it shouldn't have — in product/UX terms?
- **What this design must NOT do.** Concrete "do not" statements about product behavior, navigation, or category.

If no past attempts, name **generic anti-patterns for this product type**: a learning app becoming a passive video player; a tool dashboard becoming a marketing site; a native tablet app becoming an HTML scrollytelling page; an ops tool becoming a toddler tap-toy. The point is to protect the product-category identity, not to constrain color choices.

## Roadmap at a glance

Numbered list of all phases from `roadmap.md`, with one or two lines each describing what the phase delivers. Mark the current phase explicitly with `[CURRENT]`. The design tool needs to know:
- Where the product is going (not just where it is now)
- What surfaces, panels, and primitives will land later
- What ordering imposes on visual structure

```
1. Phase 1 — [name] [CURRENT]: [one-line capability + what it leaves in place for later]
2. Phase 2 — [name]: [one-line capability]
3. Phase 3 — [name]: [one-line capability]
...
```

## Current phase scope

What THIS phase delivers concretely, derived from `requirements.md`. One or two paragraphs + a short bullet list of what the user can do at the end of this phase. Distinguished clearly from the full vision so the tool knows what to actually design now versus what to merely accommodate.

If the phase has a meaningful daily-use slice (e.g. "morning on a train: open app, check status, approve"), describe it here in prose.

## Forward-compatibility callouts

The single most important section for preventing future-phase retrofit pain. List 3–10 capabilities that are **not in this phase** but **will land in later phases** — and explicit instructions on how this phase's design must accommodate them.

Format each callout:
- **[Capability] (Phase N).** [What lands in that phase]. [How this phase must leave room — be specific about which surface, header, area, or shape must accommodate it.]

Example (from a hypothetical SDD orchestrator):
> - **The `/build` toggle (Phase 4).** A prominent toggle to start an SDD cycle from inside a workspace lands in Phase 4. Reserve a slot in the workspace chat header — don't fill it with chat-only controls. The slot should feel intentionally empty in Phase 1, not filled by something else that has to move later.

> - **Right side panel populated (Phase 4).** Phase 3 establishes empty shells; Phase 4 fills them from `/build` state. Phase 1's full-width chat must not assume the chat will always be full-width — design it as a panel that can be flanked, not a standalone page.

If a Phase N design treats later phases as if they don't exist, the retrofit pain is the design's fault, not the implementer's. This section's job is to prevent that.

## Screen groups — what each does and why

For each natural group of screens (Login, Home, Settings, Editor, etc.), write a self-contained section that answers:

- **Job.** One paragraph — what the user comes to this group of screens to do. Frame in user motivation, not feature names.
- **Why it exists in this phase.** What about the phase scope makes this group necessary now.
- **User stories served.** List the user stories from `requirements.md` that this group fulfills. Reference by number.
- **Key behaviors the design must encode.** Behavior, not visual treatment. Examples: "the Send button morphs into a Stop button while a response is generating", "the folder picker breadcrumb cannot navigate above WORKSPACE_ROOT", "queued messages persist across browser tab close." the design tool needs to know these to design correct affordances.
- **States that surprise.** States that aren't obvious from the screen name and might be missed: queue chip below input, jump-to-latest pill while scrolled up, restart banner when the underlying process dies, etc.
- **Forward-compat for this group.** Which forward-compat callouts from above apply most directly to this group.

This section grounds the design tool in the *why* of each screen, not just the *what*. Without it, designs end up correct in form but wrong in priority.

## User stories

Verbatim from `requirements.md`. Organize by the screen group that primarily fulfills each — but a story can appear under multiple groups if it crosses surfaces (e.g. multi-device sync touches both list and chat).

Format:
```
### [Screen group name]
**N. [Story title]:** As [actor], I [specific action] so that [outcome]. [Mission flow + step reference, if any.]
```

the design tool references these while designing — when a screen feels off, the question "which user story does this serve?" cuts through ambiguity.

## Copy & content

Provide the **real on-screen text** — not lorem ipsum. The design tool returns a design with real words, so the brief must supply them.

Include (or point directly to the canonical source):
- **Representative screen copy.** Key headings, labels, and body text for the main screens. Pull from the curriculum docs, requirements.md, or mission.md — do not invent.
- **Button and action labels.** The actual text on primary and secondary actions for each screen group.
- **Microcopy.** Confirmation messages, empty states, error states, helper text. These shape affordances — the design tool needs them to size and place UI elements correctly.
- **Content source pointer.** If copy lives in a canonical doc (e.g., `curriculum/unit-1-deep-sea.md`, `requirements.md`), name it and quote the most important sections inline. Do not just reference the doc without giving the tool a usable sample.

The guiding principle: the design returned should look like the real product, not a wireframe with placeholder text.

## Information architecture

The persistent UI elements across phases — what survives, what evolves. Use a simple table:

| Element | Phase 1 | Phase 2 | Phase 3 | Phase N |
|---|---|---|---|---|
| Header | A | A | A + B | A + B + C |
| Sidebar | None | None | Empty shell | Populated |
| Right panel | None | None | Empty shell | Populated |
| Bottom bar | None | A | A | A + D |
| ...etc | | | | |

Helps the design tool design a coherent **system** — not isolated screens.

## Screen checklist

Numbered list of every screen + state from `requirements.md`'s UI Requirements table. **THIS PHASE ONLY** — future phases will have their own briefs. The list is the coverage contract: every item must appear in the returned design.

```
GROUP A
1. [Screen] — [state]
2. [Screen] — [state]

GROUP B
3. [Screen] — [state]
...
```

## Coverage notes

- **Constitution constraints** that apply to all screens (responsive viewports, touch-target size minimums, text-size minimums, accessibility minimums, platform-specific attribution requirements, product rules like no-streak/no-leaderboard) — restate from `tech-stack.md`. Do NOT include theme posture (light/dark) — that is a visual decision the tool owns.
- **Design system anchors** — patterns that should survive across surfaces (e.g., a card shape that works at full-card size on a home page AND at compact-row size in a future sidebar).
- **Most-forgotten surfaces** — list-item renderers in particular tend to get designed as detail panels but missed as compact components. Call this out for any list-item that has state variants.
- **Forward-compat reminders mapped to specific screens** — call out which screens from the checklist most directly inherit the forward-compat callouts above.
