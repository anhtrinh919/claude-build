# [Feature Name] Validation

This is the test contract for `/build-review` (pipeline-review mode). Every check listed here must pass before the phase is approved.

## Automated Checks

Run these commands. Each must exit 0.

- **TypeScript:** `tsc --noEmit` — zero errors
- **Unit tests:** `[test command]` — verifies [what specifically]
- **API — [endpoint]:** `curl -sf [method] http://localhost:[port]/api/[path]` — returns [status] with `{ field: value }`
- **API — [endpoint]:** error case — `[request with bad input]` returns 400

## Manual Verification

Walk through these checks in a browser. Each is pass/fail.

**Viewport 390px (mobile):**
- [ ] [Specific screen] — [what to check and what correct looks like]
- [ ] [Core user action] completes without horizontal scroll

**Viewport 1280px (desktop):**
- [ ] [Specific screen] — [what to check]
- [ ] Navigation between [page] and [page] works

**App Shell — Phase 0 (`initial`) only. Skip this block for `feature` and `rebuild` phases; use the regression block below instead:**
- [ ] Desktop 1280px: sidebar visible, correct item highlighted as active, Cmd+B / Ctrl+B toggles collapse
- [ ] Mobile 390px: bottom nav bar visible with correct items, hamburger opens drawer, drawer dismisses on backdrop tap
- [ ] Auth: unauthenticated visit to protected route → redirected to login → login succeeds → returned to original page
- [ ] Profile dropdown: avatar click opens menu with all standard items present; Sign Out logs user out and redirects to login
- [ ] Settings: accessible via sidebar footer link AND profile dropdown; General and Security categories present
- [ ] Toast: trigger a success action → toast appears bottom-right (desktop) or top-center (mobile), auto-dismisses in 3–5s
- [ ] Skeleton: navigate to a data-fetching page → skeleton placeholder renders before data arrives, not a blank screen
- [ ] Error boundary: with network blocked, trigger a data fetch → friendly error message with "Try again" appears, no blank screen
- [ ] Empty state: navigate to a screen with no data → first-use empty state shows (illustration + headline + CTA)
- [ ] Logo / home button: clicking the logo from any page navigates back to home/dashboard

**App Shell regression — Phase 1+ (`feature` or `rebuild`). Replace the Phase 0 block above with this:**
- [ ] Navigation: all Phase 0 nav items still accessible; active state correct on new pages added this phase
- [ ] Auth: unauthenticated visits to new routes in this phase redirect to login correctly
- [ ] Toast: toast system still fires on actions added in this phase

**User flows:**
- [ ] No dead-ends: every screen in this phase is reachable from home and can return to it — no screen traps the user (the review-time backstop for `/build-spec` reconciliation check 8).
- [ ] [Step 1 → Step 2 → Step 3]: user sees [expected result]
- [ ] Empty state: navigate to [screen] with no data → user sees [message] with [action]
- [ ] Error state: [trigger condition] → user sees [specific message], not blank screen
- [ ] [Edge case]: [what to do] → [expected behavior]

## Outcome Checks

One per PRIMARY outcome on `outcome-card.md` — same numbering. Binary and demonstrable on screen by a non-technical person ("the API returns 200" violates this). `/build-review` (pipeline-review mode) grades these explicitly.

- [ ] Outcome 1: [card outcome restated] → [the on-screen signal from the card's "Success looks like" — what a person sees that proves it]
- [ ] Outcome 2: [...]
- [ ] Outcome 3: [...]

## Definition of Done

This phase is complete when ALL of the following are true:

- [ ] All automated checks pass (exit 0)
- [ ] All manual verifications pass
- [ ] All outcome checks pass — every primary outcome on the card is demonstrably delivered
- [ ] Frontend compliance check passes (handover covers all UI requirements)
- [ ] UX review passes — no blocking issues
- [ ] user explicitly approves
- [ ] Living docs updated: README status, docs/api.md, CHANGELOG.md
