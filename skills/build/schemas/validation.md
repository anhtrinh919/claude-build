# [Feature Name] Validation

This is the test contract for `/build:review` (pipeline-review mode). Every check must pass before approval. Write each check as an observable, walkable sentence — "Rename to `../evil`: rejected inline with a plain message," not "works correctly."

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

**App Shell — Phase 0 only. Skip for Phase 1+; use the regression block below.** Write one check per decision `requirements.md ## App Shell` actually states — never a check for a pattern this app did not choose. Cover, at each breakpoint the spec names:
- [ ] Navigation: the stated pattern renders, the active item is marked, and every stated section is reachable
- [ ] Auth (if the app has accounts): unauthenticated visit to a protected route → login → returned to the original page; sign-out returns to login
- [ ] Settings (if in scope): reachable by every entry point the spec names; the stated categories are present
- [ ] Each universal pattern the spec names: trigger it and confirm the stated behavior — loading shows something, an error shows a way to recover, an empty screen shows a way forward
- [ ] Home: from any screen, the stated route home works

**App Shell regression — Phase 1+. Replaces the Phase 0 block:**
- [ ] Navigation: all Phase 0 nav items still accessible; active state correct on new pages added this phase
- [ ] Auth: unauthenticated visits to new routes in this phase redirect to login correctly
- [ ] Every universal pattern from Phase 0 still fires on the surfaces added this phase

**User flows:**
- [ ] No dead-ends: every screen in this phase is reachable from home and can return to it — no screen traps the user (the review-time backstop for `/build:spec` reconciliation check 8).
- [ ] [Step 1 → Step 2 → Step 3]: user sees [expected result]
- [ ] Empty state: navigate to [screen] with no data → user sees [message] with [action]
- [ ] Error state: [trigger condition] → user sees [specific message], not blank screen
- [ ] [Edge case]: [what to do] → [expected behavior]

## Outcome Checks

One per PRIMARY outcome on `outcome-card.md` — same numbering. Binary and demonstrable on screen by a non-technical person ("the API returns 200" violates this). `/build:review` (pipeline-review mode) grades these explicitly.

- [ ] Outcome 1: [card outcome restated] → [the on-screen signal from the card's "Success looks like" — what a person sees that proves it]
- [ ] Outcome 2: [...]
- [ ] Outcome 3: [...]

## Definition of Done

This phase is complete when ALL of the following are true:

- [ ] All automated checks pass (exit 0)
- [ ] All manual verifications pass
- [ ] All outcome checks pass — every primary outcome on the card is demonstrably delivered
- [ ] Design compliance check passes (the design source covers all UI requirements)
- [ ] UX review passes — no blocking issues
- [ ] user explicitly approves
- [ ] Living docs updated: README status, docs/api.md, CHANGELOG.md
