# App-Shell Spec — the standard shell (app-shell spec reference)

The concrete specification of the standard app shell every `ui: true` project builds in **Phase 0 (Foundation)**. This is `/build-design`'s **app-shell spec reference**: `/build-spec` (phase mode) fills `requirements.md`'s `## App Shell` section and `validation.md`'s shell block **from this doc**, adapted to the app's context (labels, icons, which settings categories apply, social-login providers).

**Best of both:** the concrete measurements/patterns below are the buildable spec; each element is tagged with the [ux-rules.md](ux-rules.md) rule (`N/F/A#`) and the [gates.md](gates.md) gate it must satisfy, so the shell is traceable to the correctness bar: on `external`, `gates.md` checks it against the exported images; on `claude-code`, impeccable owns checking the shell against these same rules directly in its own live mockups, not this catalogue. Where a rule needs its full rationale, ux-rules carries it — this doc stays concrete so an injected requirement is self-contained.

**When this applies** (by phase number):
- `ui: true`, Phase 0 → **BUILD** the shell this phase.
- `ui: true`, Phase 1+ → **INHERIT**; note only shell deltas; validation adds a regression check that the shell still works.
- `ui: true`, a phase that deliberately redesigns the shell → **REDESIGN**; fill from scratch, overriding Phase 0.
- `ui: false` → skip entirely.

---

## Navigation (responsive) → N1, N3, N18–N22, A7 · gate G-NAV

| Breakpoint | Pattern | Key specs |
|---|---|---|
| Mobile < 768px (`md`) | Bottom nav bar (3–5 items) + hamburger drawer for overflow | Drawer 280px, 300ms cubic-bezier(0.4,0,0.2,1), backdrop rgba(0,0,0,0.5) |
| Tablet 768–1023px | Icon-only sidebar rail (~64px) with hover tooltips | Toggle to full sidebar on user preference |
| Desktop ≥ 1024px (`lg`) | Persistent left sidebar (never hide primary nav behind a hamburger — N3) | 256px expanded / 56px collapsed; Cmd+B / Ctrl+B toggle; state persists in `localStorage` |

**Desktop sidebar anatomy**
- **Header:** workspace name / logo → home/dashboard (**N2** — never break this link).
- **Nav items:** icon + label (never icon-only when expanded); 36px height (44px touch — **A1**); filled-pill active state = brand-color background (**N6** "you are here").
- **Section groups:** uppercase 11–12px label, collapsible; spatial grouping, not borders.
- **Footer:** `?` Help · Settings · user avatar + name (32–36px, initials fallback) → profile dropdown.
- **Collapsed state:** icon-only requires hover tooltips.

**Mobile bottom nav anatomy**
- Height 56px + `env(safe-area-inset-bottom)`; 3–5 items (**N20**); icon 24px + 12px label; active = filled icon + 64×32px pill; always visible (does not hide on scroll).
- Hamburger drawer: same items as desktop sidebar; workspace name at top; user avatar + name above logout.

**Command palette (Cmd+K):** centered modal ~560–640px. Contents: pages, recent items, actions, settings shortcuts, help. Trigger `Cmd+K` / `Ctrl+K`.

**Breadcrumbs → N5:** show only at 3+ hierarchy levels; below global nav, above page title; separator `›`; current page last, not linked, muted; mobile → parent-only or drop.

---

## Auth shell → F13–F20 · gate G-AUTH-SET

The complete set — sign up · log in · log out · password reset · error handling for each — is mandatory where accounts exist.

**Login gate**
- **Layout:** centered card 400px desktop / full-width mobile (24px padding), ~40% from top. Hold the product's design language — don't bolt on a generic centered-card that breaks character.
- **Contents:** logo (48–64px) · headline · email · password with visibility toggle (**F14** — no confirm-password field) · "Forgot password?" right-aligned below password (**F17**) · primary CTA ("Sign in", full-width) · divider "or" · social login (Google first; GitHub for dev tools / Apple for consumer) · "Don't have an account? Sign up".
- **Errors:** inline below the field; color + icon, never color alone (**F33**, **A**-use-of-color); keep field values on error (**N16**); generic message ("email or password is incorrect") — never enumerate which was wrong (**F20**).
- **Post-login redirect:** restore `returnUrl` if bounced from a protected route; else default home.

**Session**
- Short-lived access token in memory (not localStorage); long-lived refresh token in httpOnly secure cookie, `SameSite=Strict`; re-validate on `visibilitychange` to catch server-side revocations.

**Profile dropdown** (avatar 32–36px, initials fallback, top-right): `[Avatar][Name]` / email · divider · Profile/Account · Settings · Help & Support · divider · Sign out. Popover; closes on Escape / outside click; keyboard-navigable (**A4**).

**Logout → F19:** bottom of dropdown after a divider; clear tokens → call server logout (revoke refresh) → redirect `/login`; no confirmation modal for standard apps.

---

## Settings → F21–F23 · gate G-DESTRUCT (Danger Zone)

Keep settings out of the main task flow — a secondary destination (**F21**). Entry points (both must always work): sidebar footer → Settings; profile dropdown → Settings.

**Internal layout:** desktop = vertical left sidebar of categories (mirrors the app's sidebar); mobile = full-screen list → drill into subcategory.

**Minimum categories** (**F23** grouping; omit an irrelevant one with a note in requirements.md):
```
General / Profile   → Name, avatar, email, timezone, language
Security            → Password change, connected sessions
Notifications       → Email, in-app, per-channel toggles
Appearance          → Light/dark mode
[Billing]           → only if the app has paid plans
[Team / Members]    → only if multi-user
────────────────────────────────────────────
Danger Zone         → Delete account, export data — ALWAYS last, visually separated
```
**Danger Zone → F24–F28, gate G-DESTRUCT:** destructive actions guarded (confirmation and/or undo), action-labeled buttons (not Yes/No), no dangerous default, separated from benign settings. For irreversible account deletion, require a non-standard confirmation (type "DELETE").

---

## Universal UX patterns

**Toasts / notifications** — library: Sonner (React) or equivalent (shadcn's legacy `<Toast>` is deprecated). Severity/duration: Success 3–5s · Info 5s · Warning 8s + action ("Undo") · **Error persistent, no auto-dismiss**. Position: bottom-right desktop · top-center mobile. Max 1 visible; queue the rest; pause on hover. 1–2 lines; no critical info in an auto-dismissing toast. (Undo-over-confirm — **F24**.)

**Loading states → N13, gate G-STATES** — skeleton screens for data content >500ms with known layout (shimmer: linear-gradient pulse, 1.5s infinite, mirrors the real layout); spinner only for blocking actions (save/submit/auth/payment). Never a blank screen.

**Error boundaries → N16, N17, gate G-STATES** — route-level: "Something went wrong" + "Try again" + "Go to Home", never a raw stack trace; component-level: compact inline error for individual data sections; network: distinguish offline / 5xx / 404, each with its own message + primary action.

**Empty states → N14, gate G-STATES** — first-use: illustration + headline ("No projects yet") + supporting copy + primary CTA (inviting, not apologetic); search/filter empty: inline text only ("No results for '[query]'") + suggestions + "Clear filters"; inbox-zero: celebratory, no CTA. Every empty state has a way forward — never a dead-end (**N10**).

**Notifications bell** — 24×24px, top-right, left of avatar; red badge (dot or `99+`), white border; right slide-in panel ~360–400px: header (title + unread count + "Mark all as read" + close), items reverse-chronological, bold = unread, per-item "Mark as read" / "Clear".

**Help** — `?` at sidebar footer (or header cluster on topnav layouts); optional chat widget bottom-right; Cmd+K includes help results.

**The five system states are non-negotiable for any data screen (N13–N17, G-STATES): Loading · Empty · Error · Not-found · Offline.** The shell provides the boundaries; each screen fills them.

---

## Key measurements reference

| Element | Value | | Element | Value |
|---|---|---|---|---|
| Sidebar expanded | 256px | | Toast: success | 3–5s |
| Sidebar collapsed | 56px | | Toast: error | Persistent |
| Mobile drawer | 280px | | Toast: desktop pos | Bottom-right |
| Drawer animation | 300ms cubic-bezier(0.4,0,0.2,1) | | Toast: mobile pos | Top-center |
| Sidebar item height | 36px mouse / 44px touch | | Profile avatar | 32–36px |
| Bottom nav height | 56px + safe-area-inset | | Notification panel | 360–400px |
| Bottom nav items | 3–5 | | Command palette | 560–640px + Cmd/Ctrl+K |
| Icon size | 24px | | Skeleton shimmer | 1.5s infinite |
| Touch target min | 44×44px (A1) | | Breakpoints | md 768 / lg 1024 |

---

## Sources
Material Design 3, Apple HIG, NN/g (nav/breadcrumb/empty-state studies), WCAG 2.2, Linear, Vercel, Notion sidebar, shadcn/ui Sidebar + Sonner, PatternFly notification drawer, Intercom placement, Authgear login UX, Smashing Magazine back-button UX, Tailwind responsive docs. (Rule/gate tags map to [ux-rules.md](ux-rules.md) and [gates.md](gates.md).)
