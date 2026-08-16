# Phase Outcome Card — schema

> **Conciseness.** ASD-STE100 English. Max 20 words per sentence. Max 2 sentences per entry.

File: `specs/YYYY-MM-DD-<feature-slug>/outcome-card.md`

The user-facing contract for a phase, written by `/build:spec` (phase mode). The user approves it via AskUserQuestion before any spec is written — the only artifact approved at spec time.

Plain user language — no endpoints, no schemas, no component names.

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
<Explicit exclusions, named and concrete, this phase only — "no search over the list,"
not "no advanced features": things a reasonable person might assume are included but aren't.>

## Success looks like
<One line per PRIMARY outcome: the recognizable on-screen signal that it worked.
"After saving, the order shows in the list with a green 'sent' badge".>
1. <signal for outcome 1>
2. <signal for outcome 2 — if present>
3. <signal for outcome 3 — if present>
```

Rules:
- Primary outcomes map 1:1 to the `/build:spec` primary-flow stories.
- Every "Success looks like" line must be verifiable by a non-technical person looking at a screen. "The API returns 200" is a violation.
- Card changes after approval restart `/build:spec` (phase mode) — the card is the freeze point.
