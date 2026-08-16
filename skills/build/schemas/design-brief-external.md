# Design Brief — External Tool Path

> **Conciseness.** ASD-STE100 English. Max 20 words per sentence. Max 2 sentences per entry.

Brief for handoff to whatever external design tool the user chooses. The tool owns the **entire visual look** — style, color, typography, layout, components, spacing, motion. This brief supplies **context, spec, and copy only**, so the tool knows the *why* behind every screen group and does not design this phase into a corner that later phases must retrofit around.

**Length target:** 15–40K tokens — the lower bound for small phases, the upper for foundation phases with rich product context. Stay under 50K, so the design tool keeps context budget for its own work.

**Read before writing:** `mission.md`, `product.md`, `tech-stack.md`, `roadmap.md`, `specs/YYYY-MM-DD-[feature]/requirements.md`, and any project-level `CLAUDE.md` or `past-lives.md`. Pull from them; do not paraphrase from memory.

**Do NOT include:** visual style direction, color or palette intent, typography intent, mood as aesthetic, aesthetic references, theme posture (light/dark), component names, layout hierarchy, interaction patterns, breakpoints, padding or spacing values. The tool decides the entire look; prescribing it defeats the point of the external mode. Behavioral *intent* is fine — "the empty state should feel confident, not hand-holding."

Every section below is required unless marked optional.

---

## Design intent

One short paragraph: what a user should remember or feel after using this. The experience goal, not a visual instruction. Quote the user verbatim where possible. End with *"Every design decision should be weighed against this."*

## Product type & mental model

1. **Concrete product type, one line.** "A production native tablet app", "a CLI tool", "a data dashboard web app", "a marketing site", "a B2B SaaS web app". The more specific, the better — this tells the tool which platform conventions to draw from.

2. **Which mental model to apply, and which to avoid.** AI design tools default to an interactive HTML artifact — a scrollytelling page, a slide deck, a visual playground — unless told otherwise, whatever the brief says elsewhere. Be explicit. For a native app: *"Design this as a real native application using standard platform app UI patterns — navigation, screens, touch targets, system chrome. Do NOT design it as an interactive HTML document, a scrollytelling page, a slide deck, or a code artifact."* Adapt to the real product type: for a marketing site the HTML-document model is correct; for a tablet app it is wrong.

## Product context — full vision

The full picture of what the product becomes at the end of the roadmap. Aim for **3–6 paragraphs**. Cover:

- **What it is.** Product type, deployment surface, who runs it, who uses it.
- **What it replaces.** The tools or workflows the user is moving away from.
- **What it explicitly is not.** Adjacent products that are not the target — this stops the tool pattern-matching to the wrong reference.
- **The end-state form factor.** The eventual layout, navigation model, and key surfaces, in prose. The tool designs this phase better when it can picture the destination.
- **The operator's day.** Optional, high value — a short narrative of a typical day using the product, so the tool sees screens as connected nodes in a flow.

Pull from `mission.md` and `roadmap.md`. Quote where useful.

## Patterns to avoid

Product-category and mental-model anti-patterns — wrong-category thinking the design must not fall into. Not a list of visual bans.

Where the project has predecessors — failed attempts, cousin projects, prior prototypes — name the product-level failure modes from `past-lives.md`, `CLAUDE.md`, or any lessons-learned doc. Per anti-pattern: name the category it is being confused with, what the design became that it should not have, and the concrete "do not" statements about behavior, navigation, or category.

With no past attempts, name generic anti-patterns for this product type: a learning app becoming a passive video player, a tool dashboard becoming a marketing site, a native tablet app becoming a scrollytelling page.

## Roadmap at a glance

Every phase from `roadmap.md`, one or two lines each, with the current phase marked `[CURRENT]`. This tells the tool where the product is going, which surfaces and primitives land later, and what the ordering imposes on visual structure.

```
1. Phase 1 — [name] [CURRENT]: [one-line capability + what it leaves in place for later]
2. Phase 2 — [name]: [one-line capability]
...
```

## Current phase scope

What THIS phase delivers, from `requirements.md`: one or two paragraphs plus a short list of what the user can do at the end of it. Keep it clearly distinct from the full vision, so the tool knows what to design now versus what to merely accommodate. Where the phase has a meaningful daily-use slice, describe it in prose.

## Forward-compatibility callouts

The most important section for preventing retrofit pain. List 3–10 capabilities **not in this phase** that **land in later phases**, with explicit instructions on how this phase's design must accommodate each.

- **[Capability] (Phase N).** [What lands in that phase]. [How this phase must leave room — name the surface, header, area, or shape.]

> - **The `/build` toggle (Phase 4).** A prominent toggle to start an SDD cycle from inside a workspace lands in Phase 4. Reserve a slot in the workspace chat header; do not fill it with chat-only controls. The slot should feel intentionally empty now, not filled by something that has to move later.

## Screen groups — what each does and why

For each natural group of screens (Login, Home, Settings, Editor), write a self-contained section:

- **Job.** One paragraph — what the user comes to this group to do, in user motivation, not feature names.
- **Why it exists in this phase.** What about the phase scope makes it necessary now.
- **User stories served.** By number, from `requirements.md`.
- **Key behaviors the design must encode.** Behavior, not visual treatment: "the Send button morphs into a Stop button while a response generates", "queued messages persist across a tab close". The tool needs these to design correct affordances.
- **States that surprise.** States not obvious from the screen name: a queue chip below the input, a jump-to-latest pill while scrolled up, a restart banner when the underlying process dies.
- **Forward-compat for this group.** Which callouts above apply most directly here.

## User stories

Verbatim from `requirements.md`, organized by the screen group that primarily fulfills each. A story may appear under several groups where it crosses surfaces.

```
### [Screen group name]
**N. [Story title]:** As [actor], I [specific action] so that [outcome]. [Mission flow + step reference, if any.]
```

## Copy & content

The **real on-screen text**, never lorem ipsum — the tool returns a design with real words, so the brief must supply them.

- **Representative screen copy.** Key headings, labels, and body text for the main screens. Pull from the canonical docs; do not invent.
- **Button and action labels.** The actual text on primary and secondary actions per screen group.
- **Microcopy.** Confirmations, empty states, error states, helper text. These size and place UI elements.
- **Content source pointer.** Where copy lives in a canonical doc, name it *and* quote the most important sections inline. A bare reference is not enough.

## Information architecture

The persistent UI elements across phases — what survives, what evolves — so the tool designs a coherent system rather than isolated screens.

| Element | Phase 1 | Phase 2 | Phase 3 | Phase N |
|---|---|---|---|---|
| Header | A | A | A + B | A + B + C |
| Sidebar | None | None | Empty shell | Populated |
| Right panel | None | None | Empty shell | Populated |
| Bottom bar | None | A | A | A + D |

## Screen checklist

Every screen and state from `requirements.md`'s UI Requirements table, numbered. **THIS PHASE ONLY.** The list is the coverage contract: every item must appear in the returned design.

```
GROUP A
1. [Screen] — [state]
2. [Screen] — [state]

GROUP B
3. [Screen] — [state]
```

## Coverage notes

- **Constitution constraints** that apply to every screen — responsive viewports, touch-target and text-size minimums, accessibility minimums, platform attribution requirements, product rules. Restate from `tech-stack.md`. Never include theme posture; the tool owns that.
- **Design system anchors** — patterns that must survive across surfaces, such as a card shape that works full-size on a home page and as a compact row in a future sidebar.
- **Most-forgotten surfaces** — list-item renderers get designed as detail panels and missed as compact components. Call out any list item with state variants.
- **Forward-compat reminders mapped to screens** — which checklist screens most directly inherit the callouts above.
