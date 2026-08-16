# Design tokens — the standard `design-tokens.css` must meet

> **Conciseness.** ASD-STE100 English. Max 20 words per sentence. Max 2 sentences per entry.

Read this before you write `specs/<phase>/design-tokens.css`. It applies to both design modes. Backend imports this one file and treats it as the only source for color, font, and spacing, so a token that is wrong here is wrong everywhere.

## Three tiers

1. **Primitive** — the raw value, no meaning: `blue-500`, `space-4: 16px`.
2. **Semantic** — the meaning: `color-primary`, `color-text-secondary`, `surface-error`, `spacing-inset-lg`. Theme against this layer.
3. **Component** — a per-component override that references a semantic token.

A component references a semantic token. A semantic token references a primitive. Change one primitive and it cascades. Never put a raw hex or an arbitrary px value in component markup.

## Spacing and type

- **Spacing:** 4px base, stepped in multiples of 8 — `4 · 8 · 12 · 16 · 24 · 32 · 40 · 48`.
- **Type:** 16px base × one ratio per step. 1.25 (major third) gives `14 · 16 · 20 · 25 · 31 · 39`. Body `line-height` ~1.5.

## Color

Define roles, never raw hexes: `primary`/`on-primary`, `surface`/`on-surface`, `background`, and the status set `success`/`warning`/`error`/`info` — each status with its own `on-` foreground.

Bake contrast into the pairs. Every text-on-surface combination must clear 4.5:1 before a screen uses it (3:1 for large text and UI boundaries).

For light and dark, define a parallel semantic set. Do not invert the light set: dark surfaces are dark gray, not pure black, and saturated fills come down. Re-check contrast in both.
