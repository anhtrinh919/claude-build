---
name: shape
description: 'Front gate for a new /build project. Settles the product''s shape before the constitution: a concept interview (the outcome, the user, the big either/or forks), then 3C research into customer, category, competitors. Hands the shape to /build:spec. Trigger on /build:shape or "shape this idea".'
user-invocable: true
argument-hint: "[the idea]"
---

# /build:shape — settle the product's shape

Owns the front of a new project: what the product is, who it serves, and the big either/or forks. Settle them while they are cheap to change, so `/build:spec` drills the constitution *inside* a settled shape instead of deriving the shape by interrogation.

Enter through `/build` — never off the state file directly (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`). Plain language throughout; the user is not a developer (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Mode | Model | Mechanism | Reads | Writes | Terminal step |
|---|---|---|---|---|---|
| single | Opus | inline — spawns 1 Sonnet research leaf | the idea | **Product Shape** (prose, no file) + `research.md` at the project root + `.build-state.json` (first written here) | **`shape-complete`** — written at the end of Step 1.2 |

Two steps, each its own `currentSubStep`. Builds no foundation. **No user gate** — the constitution boundary is next, so hand straight back and let the orchestrator continue in the same turn.

## Step 1.1 — Concept interview

Write `.build-state.json` — `{ stack: "build", step: "shaping-in-progress", currentSubStep: "shape.1.1" }` — before the first question. This is the project's first state write. `stack` is written here and never again; without it a later resume loads the wrong orchestrator.

Drill with `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/drilling-discipline.md` in **`concept-shape`** mode. Usually shallow — most ideas resolve at root, trunk, and branch. A shallow tree is correct here, not incomplete; leaf depth belongs to Step 1.4.

- **Root — the outcome.** Why this exists, answerable independent of any feature.
- **Trunk — who needs it most.** The actual person, not a segment ("enterprises" and "marketing teams" are filters, not people). Skip if the idea already names someone specific enough that nothing forks on it.
- **Branch — the big either/or decisions** the idea has not already settled, from this closed vocab: **single-vs-multi** (one workspace or many) · **spine** (the one core job, and which of two plausible spines) · **boundary** (a tool, a content surface, or a pipeline). Each fork must be two genuinely different products, never a fake choice. Surface via `AskUserQuestion` with the options, a plain-language tradeoff for each, and a recommended default. Two to five forks.
- **Leaf** — only where an already-settled branch is still genuinely ambiguous.

Assemble the resolved tree:

```
**Product Shape**
- **What it is:** [one line — the product as a recognizable form, reflecting the fork picks]
- **Who it's for:** [the one primary user]
- **Core job:** [the single thing it must do well — the spine]
- **Forks chosen:** [each fork, one line: `fork → choice`, plus what it rejected. /build:spec logs the rejected options to docs/rejected.md, so they are never re-offered.]
- **Beyond the core spine (deferred, not vetoed):** [2–4 things intentionally outside the core job for now; each: what it is, and why it waits. Never "cut from v1" — this is open growth, not a permanent exclusion set.]
```

Auto-move to 1.2 with a one-line transition: "Concept settled: [one line]. Researching the landscape next."

## Step 1.2 — 3C research

Write `currentSubStep: "shape.1.2"`.

**Customer** — no agent. Distil 1.1's answers into one paragraph on who this serves and what they need, and write it into `research.md ## Customer` yourself.

**Category + Competitor** — ONE Sonnet leaf, context-isolated (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md`). Verify `research.md` exists on return.

```
Research this Product Shape: [paste the shape]. Two parts, both in your return.

Part A — Category norms: 3-5 durable, specific bullets on how products of this kind typically work —
business model, must-have features, distribution and pricing norms. Ground each in a real example, not
generic wisdom.

Part B — Competitors: 3–5 products comparable to this Product Shape. Include at least one well-regarded
and at least one that failed or has known problems; mix direct competitors with adjacent products sharing
the same core user job. For each: concept (what it is, who for, core value prop), key user-facing
features, notable UI/navigation patterns worth adopting or avoiding, and stack/architecture if open-source.

Use the Mobbin MCP for each competitor's real screen-by-screen flows, plus WebSearch and WebFetch —
prefer official sites, GitHub, Product Hunt, and long-form product retrospectives. Then extract 5–10
durable, specific learnings from Part B, each tagged To-do (worth adopting) or NOT-to-do (known failure
mode). Write {project_root}/research.md in exactly this format:

# Market Research
## Customer
[left blank — the orchestrator fills this in]
## Category
- …  (3-5 bullets from Part A)
## Competitors Studied
### [Product]
- **Concept:** …
- **Key features:** …
- **UI patterns:** …
- **Codebase/Tech:** …  (omit if unavailable)
## Learnings
### To-do
- …
### NOT to-do
- …

Then return a 3–5 sentence summary of the landscape plus the top 2–3 learnings. research.md is a
reference doc, not a spec — observations only, never requirements.
```

Close the brief with the containment string from `subagent-policy.md`.

On a clean return, write `{ step: "shape-complete", currentSubStep: null }`.

## Handoff

Convey in plain language: the shape is settled, the landscape is researched, and next is the deeper product interview and the product's story for approval. Hand the **Product Shape** prose and `research.md` to **/build:spec constitution mode**.

## Ground rules

1. Two callers: `/build` on a new project, or the user directly on an idea. Never run this on your own ideas during another task.
2. This skill does not judge whether the idea is worth building — it gives the idea a shape. The user decides what to build.
3. Every big-shape fork is the user's call, surfaced as options with tradeoffs (`voice.md`). Never pick one silently.
