# UX Rules — the structural correctness ruleset

The checkable half of the skill. Most "unintuitive flow" is **structural, not aesthetic**: a screen with no purpose, a screen you can enter but not leave, a form that punishes you, a path that quietly ends. Those are mechanical defects with mechanical fixes. Rule IDs (`N#`, `F#`, `A#`) map to gate IDs in [gates.md](gates.md).

Rules are tagged `[all]`, `[web]`, `[mobile]`, `[iOS]`, `[Android]`, `[desktop]` — apply the ones that fit the project's stack (`tech-stack.md`).

---

## 1. Navigation & no-dead-ends

**Persistent navigation**
- **N1 `[all]`** Every screen except home has a visible, standard **back affordance** (nav back button on mobile; back link / breadcrumb / ancestor nav on web — never rely on the browser back button alone).
- **N2 `[web]`** The logo (top-left) links to `/`. Keep it left-aligned — centered/right logos measurably hurt homepage-finding.
- **N3 `[web]`** Persistent, visible global nav on desktop. Do **not** hide primary nav behind a hamburger — it roughly halves discoverability, worse on desktop. Inline nav at desktop widths; hamburger only for narrow/mobile.
- **N6 `[all]`** Always answer "You are here": highlight the active nav item, show a screen title, breadcrumbs where relevant.

**Route / screen architecture**
- **N8 `[all]`** Every screen reachable by a stable URL / deep link.
- **N9 `[web]`** Shared chrome (nav, sidebar) persists across child routes via a layout — put persistent nav in the layout, page content in the child.

**No dead-end screens (the core rule)**
- **N10 `[all]`** Every screen offers **at least one forward action AND one backward exit**. A detail screen links onward (related items / primary CTA) and back; a 404 links home; an empty state links to the action that fills it. This is the "emergency exit" of user control & freedom.
- **N11 `[web]`** A catch-all route returns a real 404 (not a silent 200).

**Required system states — ship all five for any data screen**
`Loading · Empty · Error · 404/Not-found · Offline.`
- **N13 `[all]`** Never a blank screen while loading or empty — users can't tell "loading" from "broken." Skeleton/status while loading; a purposeful empty state when there's genuinely no content.
- **N14 `[all]`** Empty states (a) explain what would appear here and (b) give a direct CTA to populate it.
- **N16 `[all]`** Error messages are visible, constructive, and preserve the user's input — say what went wrong and how to recover; never blame the user or discard their work.
- **N17 `[all]`** Offline/network failure is a first-class state with a distinct message + retry.

**Platform conventions**
- **N18 `[web]`** Top-nav for a few shallow sections; left sidebar for many sections / deep hierarchy; hamburger only at narrow widths.
- **N19–N22 `[mobile]`** Tab bar / nav bar only for top-level peer sections — not actions, not deep hierarchy. Android: 3–5 destinations, rail beyond five, never rail+bar together. iOS: ~5 tabs (overflow → "More"). Adapt to window size — bar <600dp, rail ≥600dp.
- **N24 `[iOS]/[desktop]`** On large canvases (iPad/desktop) prefer a sidebar in a split view over a bottom tab bar.
- **N25 `[desktop]`** Use platform-standard window chrome / menus; don't reinvent them.

---

## 2. Flows & information architecture

**Every screen earns its place (Jobs-to-be-Done)**
- **F1 `[all]`** One primary action per screen. If a stranger who can't read the language can't guess the primary action from visual hierarchy alone, the screen fails.
- **F2 `[all]`** Never let two actions compete for equal visual weight — one primary (large, colored), others secondary (smaller, muted). If two genuinely can't be ranked, split into two screens.
- **F3 `[all]`** Write a job statement for every screen: *"The user is here to ___, and their next step is ___."* Can't fill both blanks → it's an **orphan screen**: give it a next step or delete it.
- **F4 `[all]`** Break complex jobs into one-decision-per-screen sequences (the TurboTax pattern), not one dumping-ground page.
- **F5 `[all]`** If different users need different things, branch *before* the screen (route them), not on it.

**Wayfinding**
- **F6 `[all]`** No dead ends at the flow level: confirmation pages, About pages, error pages, empty states are stepping stones, each needing a next action.
- **F8 `[all]`** Labels with strong "information scent" — nav/link labels clearly predict their destination; avoid clever or vague labels.

**Onboarding**
- **F9 `[all]`** Default to *no* onboarding wall — let users reach core value first, teach in context.
- **F10 `[all]`** Onboarding is always skippable, with a visible Skip control + progress indicator.
- **F11 `[all]`** Progressive disclosure — primary options by default, advanced/rare deferred to a secondary screen.
- **F12 `[all]`** Don't gate exploration behind sign-up; offer guest access / guest checkout.

**Authentication — the complete set most apps need**
`Sign up · Log in · Log out · Password reset · error handling for each.`
- **F13** Ask for the minimum (ideally email + password); collect the rest later in profile.
- **F14** No confirm-password / confirm-email fields; offer a "Show password" toggle instead.
- **F15** Disclose password constraints up front; strength meter with real-time feedback (the one field to validate live).
- **F17** Always a visible "Forgot password?"; reset works on any device.
- **F19** Obvious log-out; clarify what signing out does.
- **F20** Security-sensitive errors stay generic ("email or password is incorrect"); reset confirms without leaking whether an account exists.

**Settings & profile**
- **F21 `[all]`** Keep settings out of the main task flow — a secondary, low-frequency destination.
- **F22 `[all]`** Sensible defaults so most users never open Settings.
- **F23 `[all]`** Standard groups: Account & profile · Notifications · Privacy & data · Appearance · Subscriptions/billing · Help/legal · Account actions (log out, delete). Separate destructive actions.

**Confirmation, undo & destructive actions**
- **F24 `[all]`** Prevention > recovery, and **undo > confirmation** — offer an Undo window (Gmail-style toast) and/or soft-delete.
- **F25 `[all]`** Confirmation dialogs only for serious/irreversible consequences; overuse trains blind "Yes."
- **F26 `[all]`** Be specific — never "Are you sure?"; restate exactly what happens ("Delete 'Q3 Budget.xlsx'? This can't be undone").
- **F27 `[all]`** Label buttons with the action ("Delete file" / "Keep file"), not Yes/No; don't default to the dangerous option.
- **F28 `[all]`** Most dangerous, irreversible actions require a non-standard gesture (type "DELETE"); keep destructive options away from benign ones.

**Forms (where flows most often break)**
- **F29 `[all]`** Group related fields with white space; visible, **top-aligned labels**.
- **F30 `[all]`** Never use placeholder text as the label — it disappears on typing, hurts memory + accessibility.
- **F31 `[all]`** Mark the minority — if most fields are required, mark only the optional ones (and vice versa).
- **F32 `[all]`** Validate inline **after the user leaves the field (on blur)**, not while typing — mid-typing errors feel like scolding. Exception: validate *live* for new passwords. Never show errors on untouched fields.
- **F33 `[all]`** Each error message sits right next to its field; red **+ an icon** (never color alone).
- **F34 `[all]`** Match input type to data so the right mobile keyboard appears (`type="email"`, `tel`, `number`, date pickers); accept forgiving formats (spaces/dashes); enable autofill; cut every non-essential field.

---

## 3. Interaction & accessibility (these become the a11y gates — axe at build time, vision estimates at design time)

**Touch/click target sizing** — default to the largest that fits.
- **A1 `[all]`** Anything tappable is **≥ 44×44 px/pt (48dp on Android)**. The WCAG 24px floor is a compliance backstop, not a design goal.

**Focus, keyboard, structure**
- **A2 `[all]`** Every keyboard-focusable element shows a visible focus indicator — `:focus-visible`, a 2px+ ring at ≥3:1 contrast. Never `outline:none` without a replacement.
- **A3 `[web]`** The focused element must not be hidden by sticky headers/footers — offset with `scroll-margin`.
- **A4 `[all]`** Everything interactive works by keyboard alone. Use native elements (`<button>`, `<a>`) so this comes free; avoid `tabindex > 0`; tab order = visual order.
- **A5 `[web]`** Provide a "Skip to main content" link.

**WCAG basics every app meets**

| Requirement | Rule | Threshold |
|---|---|---|
| Text contrast (AA) | normal text vs background | **≥ 4.5:1** |
| Large text (≥18pt / ≥14pt bold) | relaxed | **≥ 3:1** |
| Non-text contrast (AA) | UI boundaries, icons, states | **≥ 3:1** |
| Resize text (AA) | scalable without loss | up to **200%** |
| Name/Role/Value (A) | every control exposes name+role+state | native or correct ARIA |
| Non-text content (A) | meaningful images get `alt`; decorative get `alt=""` | — |
| Use of Color (A) | never convey info by color alone | pair with text/icon/pattern |

**Feedback, affordances, responsiveness, motion**
- **A6 `[all]`** Make interactivity obvious: buttons get fill/border + padding + hover/pressed/focus states; links distinguishable by more than hue. Every action needs a visible response (hover, pressed, loading skeleton/spinner, disabled at reduced opacity + `aria-disabled`, success/error). On touch there's no hover — give immediate visual feedback.
- **A7 `[all]`** Design **mobile-first** (`min-width`). Tailwind breakpoints: `sm`640 · `md`768 · `lg`1024 · `xl`1280 · `2xl`1536. Material window classes (dp): Compact <600 · Medium 600–839 · Expanded 840–1199.
- **A8 `[all]`** Honor `prefers-reduced-motion: reduce` — drop parallax, large scaling/panning, autoplay. Most UI transitions land at **150–300ms** ease-out (`cubic-bezier(0.2,0,0,1)`); hovers ~100–150ms.

---

## 4. Codeable visual system (the token discipline — feeds G-TOKENS)

**Design tokens — three tiers**
1. **Primitive/global** — raw values: `blue-500`, `space-4=16px` (no meaning).
2. **Semantic/alias** — meaning-carrying: `color-primary`, `color-text-secondary`, `surface-error`, `spacing-inset-lg`. **This is the layer you theme against.**
3. **Component** — per-component overrides referencing semantic tokens.

Rule: components reference semantic tokens; semantic tokens reference primitives. Change one primitive → it cascades. **No raw hex or arbitrary px in component markup** — that's the G-TOKENS gate.

**Spacing & type**
- **Spacing:** 4px base, step in multiples of 8 (the 8pt grid): `4·8·12·16·24·32·40·48`.
- **Type:** base 16px × a ratio per step; 1.25 (major third) → `14·16·20·25·31·39`; body `line-height` ~1.5.

**Color**
- Define semantic **roles**, not raw hexes: `primary`/`on-primary`, `surface`/`on-surface`, `background`, status `success`/`warning`/`error`/`info` (each with an `on-` foreground).
- Bake contrast into token pairs so every text-on-surface combo pre-clears 4.5:1 (3:1 large / UI boundaries).
- Light/dark: don't invert — define a parallel semantic set (dark surfaces are dark gray, not pure black; reduce saturated fills) and re-check contrast in both.

---

## Pre-ship structural checklist

- [ ] Every screen maps to one job and one primary action (F1, F3)
- [ ] No two actions have equal visual weight (F2)
- [ ] Every screen has a forward path and a back/exit (N10, F6)
- [ ] Every screen is reachable from a start point
- [ ] All error, empty, loading, not-found, offline states designed — none dead-end (N13–N17)
- [ ] Main nav visible; user always knows "you are here" (N3, N6)
- [ ] Auth set complete where accounts exist: sign up, log in, log out, forgot-password + error states (F13–F20)
- [ ] Destructive actions have specific confirmation and/or undo, no dangerous default (F24–F28)
- [ ] Forms: visible labels, inline-on-blur validation, field-adjacent errors, correct input types (F29–F34)
- [ ] Onboarding skippable, not blocking core value (F9–F12)
