# Toolkit — eyes, assets, and component defaults

Claude Code can't natively see its own rendered UI and can't draw. This file closes both gaps with tooling you already have — **no new browser/test framework** (the user's setup forbids scaffolding Playwright/Cypress/Puppeteer-npm on your own initiative).

**Prefer `/browse` (gstack) as the eyes** — it is session-isolated. The puppeteer MCP is a fallback, but it is **not isolated across concurrent agents**: when several design runs share it, it returns *stale screenshots from another session*. If you must use it, **verify `location.href` matches the mockup before trusting any capture** (create-tab → navigate → check URL → screenshot → re-check URL). This bit every parallel run during eval and is the top source of "the screenshot doesn't match what I built."

---

## Eyes — the vision loop (the most important part)

The loop a front-end developer runs without thinking: **write → render → look → adjust → repeat.** Make it explicit.

1. **Render** the mockup — open the HTML locally (`open <file>` on macOS) or serve the mockups dir (`python3 -m http.server` in `mockups/`, then navigate to the file).
2. **Screenshot** with the browser tool you have — `/browse` or the puppeteer MCP — to `specs/<phase>/mockups/.review/<name>.png`. Use `browser_resize` / a viewport arg to shoot at **390 / 768 / 1280px** for `G-RESPONSIVE`.
3. **Look and check** — you are vision-capable; open the PNG and judge it against the visual gates: is it *too busy / balanced / too sparse* (`G-DENSITY`), and any concrete defects — *overlap/collision · clipping/cut-off · broken image · empty control · >1 equal-weight primary CTA* (`G-DEFECTS`). Name the concrete issues. (Holistic "is the hierarchy right / what do I look at first" is the human's call at phase-end review — not scored here.)
4. **Fix, re-shoot.** Repeat until only nitpicks remain. **The user sees the second draft, never the first** — self-critique is what separates this from "render once and hope."

**Compare-to-reference (highest-leverage when a target exists):** put the target image in the repo, render the mockup, screenshot, and enumerate every visual difference vs `reference.png` → fix → repeat until they match. Turns "make it look like this" into a converge-to-target loop (gate `G-REFERENCE`).

### Running axe-core on a static mockup (gate G-A11Y-AXE/TARGET/FOCUS)

The mockups are real HTML, so axe is fully valid. Inject it via the browser tool — no framework install:

- Load the mockup, then evaluate in-page: fetch the axe-core UMD build from a CDN (or a local `node_modules/axe-core/axe.min.js` if the project already has it), then `await axe.run()`.
- With the puppeteer MCP: `puppeteer_navigate` to the mockup → `puppeteer_evaluate` a snippet that appends the axe `<script>`, waits for `window.axe`, and returns `JSON.stringify((await axe.run()).violations)`.
- With `/browse`: navigate to the mockup and run the equivalent page script.
- Save the violations JSON to `.review/axe.json`; **0 violations** (or justified exceptions) is the pass. Also surfaces missing labels (G-FORMS) and undersized targets (with the target-size rule enabled).

---

## Free asset recipes (no keys — first choice)

These need no external service and are fully deterministic.

- **Hand-written SVG — icons, logos, badges, progress rings, wordmarks.** SVG is plain XML; type and edit it directly (recolor, resize, swap text). First choice for geometric marks. Not for photorealism/faces.
- **Mermaid — diagrams and the Milestone-1 wireflow.** Write a Mermaid definition, render to SVG/PNG via `mmdc` (mermaid-cli) or a Mermaid live-render in the browser. This *is* how the structure map gets rendered for the Show Rule.
- **Icon libraries (npm, MIT):** **Lucide** (`lucide-react` — the safe default), **Heroicons**, **Phosphor**, **Tabler**. For breadth, **Iconify** (`@iconify/react`) — 275k+ icons from 200+ sets under one API. Prefer these over generating icons.
- **@vercel/og — data-driven images** (social cards, og:images, certificates, receipts, report covers): HTML+CSS → SVG → PNG, no browser. The free replacement for Canva autofill.
- **Real charts (never fake chart images):** **matplotlib** or **Vega-Lite** locally, or **QuickChart** (post a Chart.js JSON config → PNG URL, free tier). Render true charts from real numbers — never draw a fake chart in SVG.
- **Placeholders:** `placehold.co` (keyless). For real photos in a mockup, a specific relevant image beats a gray box — but never ship Unsplash *links* as the final asset (craft ban).

---

## Optional raster generation (env-key-gated — only when free recipes can't)

For real photos, branded illustrations, or custom raster icons the free recipes can't produce. Each needs an API key the user supplies for **that service** (these are third-party image APIs, unrelated to the Claude subscription — do not reach for any Anthropic key here). Call via bash + the service's REST API. Skip with a logged reason if no key is set — never block the design on a missing image key.

| Need | Tool | Note |
|---|---|---|
| App icons / brand **SVG** (editable vector) | **Recraft** V4 vector API | The only major platform with true editable SVG output; a consistent icon set in one call. |
| **Legible text in an image** (poster, banner) | **Ideogram** | Best at rendering readable text. |
| Photoreal hero / 4K | **Imagen 4 / Nano Banana** (Gemini API) | Photoreal, strong text. |
| **Transparent PNG** | **gpt-image-1.5** | `background:"transparent"` works here — **not** gpt-image-2, which dropped transparency. |
| Cheap bulk backgrounds/textures | FLUX / SD via fal.ai or Replicate | ~cents/image. |

External design tools are *not* used in this track — this is the `claude-code` track (Claude designs it itself); the `external` track is the other one. The raster gap is the one thing this track can't do natively; these APIs are the fill.

---

## Component-library defaults (per platform — build on these, don't reinvent)

Accessible-by-default primitives save you most of the a11y gates for free.

**Web — React (primary recommendation):**
- **shadcn/ui + Radix + Tailwind** — copy-in source on Radix primitives; you own the code, accessible by default. The current default for new React apps.
- Icons: **Lucide** (Iconify for breadth).
- Plain HTML/CSS: default to **semantic native elements** (`<button>`, `<a>`, `<nav>`, `<main>`, `<label>`, `<dialog>`) — accessible by default, the highest-leverage a11y choice.

**Mobile — React Native:** **gluestack UI + NativeWind** (or **React Native Paper** for a Material look) · Expo Router (deep-linkable routes, `+not-found`) · React Navigation (tab `backBehavior`, predictive back).

**Mobile — Flutter:** stock **Material/Cupertino** widgets; wrap custom UI in `Semantics`.

**Desktop:** **Electron or Tauri + the web stack above** — carries web accessibility and the vision loop over unchanged.

Match the project's actual `tech-stack.md` — these are the defaults when the stack is open, not overrides when it's already chosen.

---

## Tokens

Three-tier semantic tokens (ux-rules §4). 8pt spacing, 16px×1.25 type, semantic color roles with contrast baked into the pairs. When the phase locks a visual language (Milestone 2), the tokens are the durable artifact — build-design's handover step exports them to `design-tokens.css` for backend to import. Keep raw values out of component markup (gate G-TOKENS).
