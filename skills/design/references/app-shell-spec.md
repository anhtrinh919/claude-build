# App-Shell Spec — the standard shell

The spec for the app shell every `ui: true` project builds in **Phase 0 (Foundation)**. `/build:spec` phase mode fills `requirements.md`'s `## App Shell` section and `validation.md`'s shell block from this doc, adapted to the app's context — labels, icons, which settings categories apply, social-login providers.

**When this applies** (by phase number):
- `ui: true`, Phase 0 → **BUILD** the shell this phase.
- `ui: true`, Phase 1+ → **INHERIT**; note only shell deltas; validation adds a regression check the shell still works.
- `ui: true`, a phase that deliberately redesigns the shell → **REDESIGN**; fill from scratch, overriding Phase 0.
- `ui: false` → skip entirely.

---

## Breakpoints

Mobile-first, `min-width` queries. Tailwind: `sm` 640 · `md` 768 · `lg` 1024 · `xl` 1280 · `2xl` 1536. Material window classes (dp): Compact <600 · Medium 600–839 · Expanded 840–1199.

## Navigation (responsive)

Pick by app shape: top nav for a few shallow sections; a left sidebar for many sections or a deep hierarchy; a hamburger only at narrow widths.

| Breakpoint | Pattern | Key specs |
|---|---|---|
| Mobile < 768px (`md`) | Bottom nav bar (3–5 items) + hamburger drawer for overflow | Drawer 280px, 300ms cubic-bezier(0.4,0,0.2,1), backdrop rgba(0,0,0,0.5) |
| Tablet 768–1023px | Icon-only sidebar rail (~64px) with hover tooltips | Toggle to full sidebar on user preference |
| Desktop ≥ 1024px (`lg`) | Persistent left sidebar — never hide primary nav behind a hamburger | 256px expanded / 56px collapsed; Cmd+B / Ctrl+B toggle; state persists in `localStorage` |

**Desktop sidebar anatomy**
- **Header:** workspace name / logo → home/dashboard, never broken.
- **Nav items:** icon + label (never icon-only when expanded); 36px height, 44px touch target; filled-pill active state = brand-color background.
- **Section groups:** uppercase 11–12px label, collapsible; group by spacing, not borders.
- **Footer:** `?` Help · Settings · user avatar + name (32–36px, initials fallback) → profile dropdown.
- **Collapsed:** icon-only, hover tooltips required.

**Mobile bottom nav anatomy**
- Height 56px + `env(safe-area-inset-bottom)`; 3–5 items; icon 24px + 12px label; active = filled icon + 64×32px pill; always visible, never hides on scroll.
- Hamburger drawer: same items as desktop sidebar; workspace name at top; avatar + name above logout.

**Tab-bar and rail rules.** A tab bar carries top-level peer sections only — never actions, never a deep hierarchy. Android: 3–5 destinations, switch to a rail beyond five, never both together. iOS: about 5 tabs, overflow into "More". Large canvas (iPad, desktop): prefer a sidebar split view over a bottom tab bar.

**Route architecture.** Every screen is reachable by a stable URL or deep link. Shared chrome (nav, sidebar) persists across child routes through a layout: nav in the layout, page content in the child. A catch-all route returns a real 404, never a silent 200.

**Command palette:** centered modal ~560–640px. Contents: pages, recent items, actions, settings shortcuts, help. Trigger `Cmd+K` / `Ctrl+K`.

**Breadcrumbs:** show only at 3+ hierarchy levels; below global nav, above page title; separator `›`; current page last, not linked, muted; mobile → parent-only or drop.

---

## Auth shell

The complete set — sign up · log in · log out · password reset · error handling for each — is mandatory where accounts exist.

**Login gate**
- **Layout:** centered card 400px desktop / full-width mobile (24px padding), ~40% from top. Hold the product's design language.
- **Contents:** logo (48–64px) · headline · email · password with visibility toggle, and no confirm-password field · "Forgot password?" right-aligned below password · primary CTA ("Sign in", full-width) · divider "or" · social login (Google first; GitHub for dev tools, Apple for consumer) · "Don't have an account? Sign up".
- **Fields:** ask for the minimum, ideally email and password; collect the rest later in profile. Disclose password constraints up front; validate a new password live, with a strength meter.
- **Errors:** inline below the field; color **and** an icon, never color alone; keep field values on error; generic message ("email or password is incorrect") — never enumerate which was wrong. Password reset confirms without revealing whether an account exists, and works on any device.
- **Post-login redirect:** restore `returnUrl` if bounced from a protected route, else default home.

**Session.** Short-lived access token in memory, not `localStorage`; long-lived refresh token in an httpOnly secure cookie, `SameSite=Strict`; re-validate on `visibilitychange`.

**Profile dropdown** (avatar 32–36px, initials fallback, top-right): `[Avatar][Name]` / email · divider · Profile/Account · Settings · Help & Support · divider · Sign out. Popover; closes on Escape or outside click; keyboard-navigable.

**Logout:** bottom of dropdown after a divider; clear tokens → call server logout (revoke refresh) → redirect `/login`; no confirmation modal.

---

## Onboarding

Default to **no onboarding wall** — reach core value first, teach in context. Where onboarding exists, make it skippable with a visible Skip control and progress indicator. Show primary options by default; defer advanced or rare ones to a secondary screen. Never gate exploration behind sign-up — offer guest access or guest checkout.

## Settings

Keep settings out of the main task flow — a secondary, low-frequency destination with defaults good enough most users never open it. Entry points (both must work): sidebar footer → Settings; profile dropdown → Settings.

**Internal layout:** desktop = vertical left sidebar of categories (mirrors the app's sidebar); mobile = full-screen list → drill into subcategory.

**Minimum categories** (omit an irrelevant one with a note in `requirements.md`):
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

**Danger Zone.** Prevention beats recovery; undo beats confirmation — offer an undo window or soft delete before a dialog. Use a dialog only for serious or irreversible consequences. Be specific — not "Are you sure?" but "Delete 'Q3 Budget.xlsx'? This can't be undone". Label buttons with the action ("Delete file" / "Keep file"), never Yes/No, never default to the dangerous option. For irreversible account deletion, require typing "DELETE". Keep destructive actions away from benign ones.

---

## Universal UX patterns

**Interactive affordances** — a control must look like a control: fill or border, padding, hover, pressed, focus, and disabled states. Distinguish links by more than hue. Every action gets visible response — hover, pressed, a loading skeleton or spinner, disabled at reduced opacity plus `aria-disabled`, success or error. Touch has no hover — give immediate visual feedback.

**Motion** — honor `prefers-reduced-motion: reduce`: drop parallax, large scaling/panning, autoplay. Most UI transitions land at **150–300ms** ease-out (`cubic-bezier(0.2,0,0,1)`); hovers at ~100–150ms.

**Toasts / notifications** — library: Sonner (React) or equivalent (shadcn's legacy `<Toast>` is deprecated). Severity and duration: Success 3–5s · Info 5s · Warning 8s + action ("Undo") · **Error persistent, no auto-dismiss**. Position: bottom-right desktop, top-center mobile. Max 1 visible; queue the rest; pause on hover. 1–2 lines.

**Loading states** — skeleton screens for data content over 500ms with a known layout (shimmer: linear-gradient pulse, 1.5s infinite, mirroring the real layout); a spinner only for a blocking action — save, submit, auth, payment. Never a blank screen.

**Error boundaries** — route-level: "Something went wrong" + "Try again" + "Go to Home", never a raw stack trace. Component-level: a compact inline error for the data section. Network: distinguish offline / 5xx / 404, each with its own message and primary action. An error message preserves the user's input, says how to recover, and never blames the user.

**Empty states** — first-use: illustration + headline ("No projects yet") + supporting copy + primary CTA, warm not apologetic; search or filter empty: inline text only ("No results for '[query]'") + suggestions + "Clear filters"; inbox-zero: celebratory, no CTA. Every empty state explains what would appear and gives a way to fill it.

**Notifications bell** — 24×24px, top-right, left of avatar; red badge (dot or `99+`), white border; right slide-in panel ~360–400px: header (title + unread count + "Mark all as read" + close), items reverse-chronological, bold = unread, per-item "Mark as read" / "Clear".

**Help** — `?` at the sidebar footer, or the header cluster on top-nav layouts. Cmd+K includes help results.

**The five system states are non-negotiable for any data screen: Loading · Empty · Error · Not-found · Offline.** Every screen offers at least one forward action and one backward exit — a detail screen links onward and back, a 404 links home, an empty state links to the action that fills it.
