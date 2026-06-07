# Phase Outcome Card — schema

File: `specs/YYYY-MM-DD-<feature-slug>/outcome-card.md`

The user-facing contract for a phase. Written by `/ba` Mode 2 from the drilling session; approved by the user via AskUserQuestion BEFORE any spec is written. This is the ONLY artifact the user approves at spec time — requirements.md / plan.md / validation.md are machine-validated against this card and auto-proceed.

The card flows end-to-end: `/spec`'s skeptic panel checks every requirement traces to it; `validation.md` carries one Outcome check per primary outcome; `/review` grades each outcome explicitly; the dogfood handoff's "What you can test" bullets are generated from it. Keep it in plain user language — no endpoints, no schemas, no component names.

```markdown
---
phase: <N>
feature: <kebab-case-slug>
approved: <YYYY-MM-DD — set the moment the user picks Approve; never pre-filled>
---

# Phase <N> — <Feature name>

## Phase goal
<One sentence: what this phase is for, in user language.>

## You'll be able to
<1–3 PRIMARY outcomes. Each observable and specific — something the user can do
at the end of this phase that they can't today. "Create an order from my phone
and see it appear on the store screen" — not "order management".>
1. <outcome>
2. <outcome — optional>
3. <outcome — optional>

## Also included
<Secondary outcomes — nice-to-haves shipping in this phase. "None" is valid.>

## Not in this phase
<Explicit exclusions, named. Things a reasonable person might assume are included
but aren't.>

## Success looks like
<One line per PRIMARY outcome: the recognizable on-screen signal that it worked.
"After saving, the order shows in the list with a green 'sent' badge" — the user
should be able to verify each line by hand during dogfood.>
1. <signal for outcome 1>
2. <signal for outcome 2 — if present>
3. <signal for outcome 3 — if present>
```

Rules:
- Primary outcomes map 1:1 to the `/ba` primary-flow stories — same count, same order.
- Every "Success looks like" line must be verifiable by a non-technical person looking at a screen. "The API returns 200" is a violation.
- Card changes after approval restart `/spec` Mode 2 — the card is the freeze point, not the spec files.
