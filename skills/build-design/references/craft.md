# Craft — the taste layer (self-contained)

Structure (ux-rules) makes a UI *work*. Craft makes it not look generic, templated, or AI-generated. This file is self-contained — it's the reason `build-design` doesn't need any other design skill loaded. Read it at Milestone 2 (Look anchor) and keep it in mind through Milestone 3.

The failure mode this prevents: a technically-correct, accessible, well-structured UI that still looks like every other AI-generated app — the purple gradient, the three equal cards, the everything-centered, the timid grays. Correct but forgettable.

---

## 1. Register — decide before you design

Every surface is one of two registers. Name it first; it changes every subsequent choice.

- **Brand** — the design *is* the product: marketing, landing, campaign, portfolio, long-form. Expressive, memorable, can be loud. Big type, motion, personality.
- **Product** — the design *serves* the product: app UI, dashboard, admin, tool, settings. Calm, legible, gets out of the way. The interface should disappear so the task is what the user sees.

Priority for deciding: (1) a cue in the task ("landing page" vs "dashboard"); (2) the surface in focus; (3) the product's stated register in `product.md` / brief. A dashboard styled like a landing page is a register error — the most common one.

---

## 2. Color — commit to a position

The single biggest anti-slop lever. Timid, desaturated, "safe" color is what makes AI UIs read as AI. Pick a **commitment level** deliberately and hold it across the whole surface:

- **Restrained** — near-monochrome, one accent used sparingly. (Most product/tool UIs.)
- **Committed** — a clear brand color carrying primary actions + accents, neutral everywhere else.
- **Full** — brand color plus a secondary, used with confidence across surfaces.
- **Drenched** — color saturates the whole surface (bold brand/marketing).

Rules of the craft:
- Work in **OKLCH**, not hex-by-feel — it keeps lightness perceptually even across hues so your palette doesn't have one color that "pops wrong."
- **Tinted neutrals, not pure gray.** Grays carry a hint of the brand hue. Pure `#888` grays are a tell.
- **Dark surfaces are dark *gray*, not pure black**; reduce saturated fills in dark mode; re-check contrast in both themes.
- Write a one-line **scene sentence** before choosing — "a calm slate workspace with a single warm amber accent for the primary action" — it forces a dark-vs-light and a commitment decision instead of defaulting.
- Every text-on-surface pair still clears the ux-rules contrast floors. Committed color and accessibility are not in tension — bake contrast into the token pairs.

---

## 3. Type — hierarchy by scale *and* weight

- One typeface family does most work; a second only with reason. **Avoid `Inter` as the reflex default** — it's the AI-UI house font; a deliberate alternative reads as intentional.
- Establish hierarchy with a **scale ratio ≥ 1.25** and **weight contrast**, not just size. A heading is bigger *and* heavier; secondary text is smaller *and* lighter/muted.
- Body `line-height` ~1.5; headings tighter. Don't center long-form body text.

---

## 4. Layout — escape the reflex

- **"Three equal cards in a row" is the lazy answer.** Vary the grid: a hero + supporting items, an asymmetric 2/3–1/3 split, a real content hierarchy. Use **CSS Grid** for structure, not flex-percentage math.
- Not everything centered. A left-aligned, content-first layout reads as more considered than the centered-everything default.
- Whitespace is structure, not filler — group with space (ties to F29). Generous, *rhythmic* spacing on the 8pt grid, not random gaps.

---

## 5. Anti-slop bans (check every screen)

Instant tells that a UI was generated, not designed. None of these ship:

- **Purple/violet gradient backgrounds** (the single most common AI tell).
- **Decorative blobs, wavy SVG dividers, floating gradient orbs.**
- **Uniform bubbly border-radius** on everything — vary radius by element role, or commit to a sharper system.
- **Everything centered** — centered hero, centered cards, centered nav, all at once.
- **Generic filler** — "Lorem ipsum," round-number fake stats ("10,000+ users"), Unsplash-link placeholder images, `<name>@example.com`. Use realistic, specific mock content.
- **`Inter` + purple glow** — the house style of the undifferentiated AI app.

---

## 6. The Trunk Test (every screen must pass)

Screenshot the screen. Within ~2 seconds, without reading carefully, can you answer:
1. **What product is this?**
2. **What page / screen am I on?**
3. **What are the major sections?**
4. **What can I do here** (what's the primary action)?

Any "I can't tell" = broken hierarchy — design *toward* a clear answer to all four; it's the taste target every layout choice above serves. But this holistic verdict is the **human's** call at phase-end review, not a model-run gate — models judge hierarchy near-randomly, so the self-critique loop verifies the decomposable proxies instead (`G-DENSITY`, `G-DEFECTS`).

---

## 7. Motion — restrained and physical

- UI transitions **150–300ms**, **ease-out** (things decelerate as they arrive — `cubic-bezier(0.2,0,0,1)` or an ease-out exponential); hovers ~100–150ms.
- Motion clarifies state change (what appeared, what moved), it doesn't decorate. No autoplay, no parallax-for-its-own-sake.
- Always honor `prefers-reduced-motion: reduce` (ties to A8). Hardware-accelerated transforms only (`transform`/`opacity`), never animate layout properties.

---

**How craft meets the forks:** color commitment, layout pattern, and register are exactly the calls the Fork Protocol renders as visual options at Milestone 2. Don't argue them in prose — build the anchor screen two ways and let the user see the difference.
