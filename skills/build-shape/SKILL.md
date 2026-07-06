---
name: build-shape
description: >
  New-project front gate for the /build SDD stack, plus a standalone idea-verdict. Two modes. (1) Shaping — auto-fired by /build at the top of every new project, before constitution: a fork-first gate that validates demand is real, surfaces the big either/or decisions (who it's for, single vs. multi, the core spine, the boundary), then assembles and pressure-tests the product's big-picture Product Shape through a seven-lens cold shower — looping until the user picks Proceed / Reshape / Drop — and finally researches 3–5 comparable products into research.md. (2) Verdict — user-invocable standalone: the full lens-by-lens table on any specific idea, feature request, or "let's add X," severity per lens, one verdict. Demand is validated ONCE here; downstream /build-spec never re-drills it. Trigger on /build-shape, "is this a bad idea", "challenge this idea", "should I build this", "shape this idea" — and automatically whenever /build starts a new project.
user-invocable: true
argument-hint: "[the idea — or a feature/change to challenge in verdict mode]"
---

# /build-shape — shape the product, or cold-shower an idea

Part of **/build**; runs **inline** in the orchestrator's session (it holds a live user gate and spawns a research leaf — either forces inline). The rest of the build stack is biased toward validating what the user asks for; this skill is the inverse. **Default stance: the idea is bad until every lens has fired and returned clean** — the bar to clear is "all lenses fired clean," never "I couldn't immediately think of an objection."

Where a quoted string is user-facing it's the intent to convey in plain language — never a path, stack name, or jargon (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Mode | Model | Mechanism | Reads | Writes | Terminal step |
|---|---|---|---|---|---|
| **shaping** (auto-fired by /build on a new project) | Opus | inline — holds the user gate; spawns 1 Sonnet research leaf | the idea (grows into a transcript as forks resolve) | **Product Shape** (prose, no file) + `research.md` at project root | none — conversation-only; `.build-state.json` is first written later by /build-spec |
| **verdict** (user-invocable standalone) | user's session model | inline | the idea text + whatever project docs exist | one lens-by-lens report in chat | none |

Hands to **/build-spec constitution mode**. No foundation is built here, no state file is written here.

**Mode detection:** called by /build on a new project (no `mission.md`) → **shaping**. Invoked directly on a specific idea → **verdict**.

---

## The seven lenses (both modes run them)

Load-bearing catalogue — the frames, attack questions, and severity thresholds are the domain signal; keep them. In **shaping** the lenses run *internally* as a pressure-test (they feed the fatal kill-check + the "kept out of the shape" list). In **verdict** they render the full table. Grounding differs: shaping skims only `~/.claude/skills/` (Lens 2) and `~/.claude/CLAUDE.md` (Lens 3) — no project docs exist yet; verdict grounds on whatever docs exist (see that mode). Earlier lenses kill faster — a Lens 1 kill means 4–7 needn't be exhaustive — but every lens fires at least briefly so the user sees what *wasn't* an objection.

**Lens 1 — Solution-as-problem framing.** Layman ideas often describe a *solution* ("add a button to do X") when the real ask is a *problem* ("I keep losing context when I switch tasks"); until the problem is named you can't evaluate the solution. Attack: what is the user actually trying to accomplish — strip the mechanism, name the friction? Is the proposed solution the *only* path or one of several (list 2–3)? Evidence they tried other approaches and ruled them out, or first idea they had? → *problem unclear, or solution one of several plausible paths without justification* = **High**, reframe before building.

**Lens 2 — Existing-solution overlap.** The thing may already exist, partially exist, or duplicate another skill/feature/library — building new when an existing thing covers 80% is bloat. Attack: does the project (or `~/.claude/skills/`) already do this or 80% of it? An existing feature extensible cheaply to cover it (extension cost vs. new-build)? An external library/service/built-in being reinvented? → *existing covers ≥80%* = **High**, extend instead; *partial 40–80%* = **Medium**, surface and let the user decide.

**Lens 3 — Internal consistency.** A new idea that contradicts a prior decision is usually (b) the new idea is wrong, rarely (a) the prior decision was — default to (b). Attack: contradicts anything explicit in mission.md ("does NOT do X") / roadmap.md ("out of scope")? Requires a tech or pattern ruled out in tech-stack.md? Contradicts a CLAUDE.md constraint or an established pattern? (New project: only `~/.claude/CLAUDE.md` global constraints exist to check against.) → *direct contradiction* = **Critical**, drop or explicitly reopen the prior decision; *soft tension* = **Medium**, surface it.

**Lens 4 — Demand reality.** "I want this" is the weakest signal — real demand is recurring friction, multiple users asking, or a measurably broken workflow; one-time itches usually shouldn't get features. Attack: when did the user last hit this friction — once, thrice, weekly? If once, how sure it recurs? Actual pain or a "nice to have" surfaced from staring at the product? Multi-user: who else asked — if only this user, is scratch-your-own-itch OK here? → *"once and not sure it recurs"* = **High**, defer until it recurs; *solo/personal project* drops one level (allowed to scratch own itch).

**Lens 5 — Cost/value sanity.** Build cost = design + implement + test + maintain forever; value = savings × frequency × users. Cheap-to-build but rarely-used is bloat; expensive but high-value is fine. Attack: rough build cost — hours/days/weeks? Frequency after shipping — daily/weekly/monthly/once? Maintenance burden — new dependency, integration point, edge-case category? Does value (frequency × per-use savings) recover build + maintenance within 6 months? → *cost > expected value over 6 months* = **High**, drop or scope smaller; high-leverage tooling amplifies value — adjust.

**Lens 6 — Bloat detection.** Features either *unlock a new use case* (good) or *add a knob to an existing capability* (almost always bloat) — every knob is cognitive overhead for every future user. Attack: unlocks a currently-impossible use case, or parameterises something fixed? If parameterising, how many use the non-default — if <10%, the knob isn't earning its cost. Adds a setting/flag/mode — justify why hardcoded won't work. Makes the surface more complex for users who don't need it? → *knob with <10% expected non-default usage* = **High**, drop or hardcode the most useful value.

**Lens 7 — One-size-fits-all.** A specific solution to the actual case beats a general framework handling that case + 10 hypothetical others. Attack: specific to the actual problem, or a general framework/pluggable architecture? Are the "general" capabilities driven by real cases or imagined ones? Could a one-off function/script solve the specific case, deferring the abstraction until 3 cases exist? → *general but only one concrete use case* = **High**, ship the specific solution and defer abstraction.

**Severity vocab (closed):** **Critical** (alone kills it) · **High** (significant; two Highs → reshape) · **Medium** (note, not fatal alone) · **Low** (minor) · **Clean** (fired, found nothing).

---

## Mode: shaping (called by /build on a new project)

The job is to settle the product's big-picture shape — what it is, who it's for, its core job, the make-or-break forks — while it's cheap to change, so /build-spec drills the constitution *inside a settled shape* instead of deriving the shape by interrogation. Three passes, each gated; the blocks below are literal output shapes.

### Pass 1 — Demand (validated ONCE, here)

> **Dedup contract:** this demand pass MOVED here from the old BA drill. **/build-spec constitution inherits validated demand and does NOT re-drill it** — it drills mission / product / stack / roadmap inside the shape. Do not duplicate this downstream.

Drill three threads via `AskUserQuestion`, taking a position on every answer ("that could work" is banned — say whether it *will* work and what evidence would change your mind; name the failure pattern):

- **Strongest evidence someone wants this** — push until you hear: someone paid, was upset when it broke, built a workflow around the problem. **Red flags, NON-demand:** "waitlist signups," "VCs excited," "people say it sounds useful."
- **What people do RIGHT NOW to solve it, even badly** — push until you hear a specific workaround, hours wasted, duct-taped tools. **Red flag:** "nothing — there's no solution" means the problem isn't painful enough.
- **Who needs this most — the actual person** — push until you hear a role, a specific consequence, something they said directly. **Red flag:** "enterprises," "marketing teams" are filters, not people.

### Pass 2 — Forks

A doomed idea shouldn't be shaped: if a lens fires **fatal** (a `~/.claude/CLAUDE.md` contradiction, ≥80% existing-skill overlap the user can't distinguish, or no problem nameable after 2–3 rounds), surface it as a blocking concern *before* any fork. Otherwise identify the 2–5 big-shape either/or decisions that set the essential form and are expensive to reverse — pick only those the idea hasn't already settled, from this closed vocab: **who** (just the user / a team / the public) · **single-vs-multi** (one workspace or shop / many) · **spine** (the one core job — and which of two plausible spines it's built around) · **boundary** (a tool / a content surface / a pipeline). Each fork is a real decision — two genuinely different products, never a fake choice — carrying its options + each option's plain-language tradeoff + a recommended default (`voice.md` fork form). Surface via `AskUserQuestion`.

```
**Idea (as I read it):** [one-sentence restatement — so a misread is caught fast]
**Blocking concern (only if a lens fired fatal):** [one sentence + which lens. Omit this line entirely if nothing is fatal.]

**Big-shape forks:**
1. **[fork name]** — [Option A] / [Option B]. Recommend **A**: [why A, what B costs — one line].
   (2–5 forks, recommended option first on each, concrete to THIS idea — never generic philosophy.)
```

### Pass 3 — Assemble the shape + gate

Assemble from the fork picks and run the lenses once more against the *assembled* product. Then gate.

```
**Product Shape**
- **What it is:** [one line — the product as a recognizable form, reflecting the fork picks]
- **Who it's for:** [the one primary user]
- **Core job:** [the single thing it must do well — the spine]
- **Forks chosen:** [each big-shape fork the user picked, one line: `fork → choice`. Carried forward; /build-spec writes these into docs/decisions.md ## User decisions when it scaffolds living docs, so they're never re-litigated.]
- **Beyond the core spine (deferred, not vetoed):** [2–4 things intentionally out of the core job *for now* — fed by the bloat / one-size / existing-overlap lenses; each: what + why it waits. Open-growth framing: these can grow in as later phases; never frame as "cut from v1" or a permanent exclusion set.]

**Holds up?** [one line: does the shaped product survive the lenses? Name the one risk to watch, or "clean — every lens cleared."]
```

Surface the Product Shape, then gate via `AskUserQuestion` — closed vocab **Proceed / Reshape / Drop**:
- **Reshape** → fold the change into the idea transcript, re-run Pass 2 + Pass 3, re-gate. Loop until Proceed or Drop.
- **Drop** → stop. No research, no handoff. (Reserve for the fatal cases above; shaping's job is to give the idea a viable form, not to veto projects.)
- **Proceed** → spawn the research leaf, then hand to /build-spec.

### Research leaf (only on Proceed)

Spawn ONE Sonnet agent, context-isolated (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md`). Verify `research.md` exists on return (Rule 5); brief:

```
Research 3–5 products comparable to this Product Shape: [paste the shape]. Include at least one well-regarded and at least one that failed or has known problems; mix direct competitors with adjacent products sharing the same core user job. For each product find: concept (what it is / who for / core value prop), key user-facing features, notable UI/navigation patterns worth adopting or avoiding, and stack/architecture if open-source (skip if unavailable). Use WebSearch + WebFetch — prefer official sites, GitHub, Product Hunt, long-form product/design retrospectives. Then extract 5–10 durable, specific learnings, each tagged To-do (worth adopting / inspired by) or NOT-to-do (known failure mode / antipattern). Write {project_root}/research.md in exactly this format:

# Market Research
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

Then return a 3–5 sentence summary of the landscape + the top 2–3 learnings. research.md is a reference doc, not a spec — observations only, never requirements. no spawning agents, no /build*, no claude -p, no commit, no server, don't address the user — return content/paths only.
```

### Handoff

Convey in plain language: the shape's locked, you'll study a few comparable products, then write the product story for approval. Hand the **Product Shape** prose to **/build-spec constitution mode**. Write no state file.

---

## Mode: verdict (user-invocable standalone)

A full lens-by-lens cold shower on one specific idea, feature request, or "let's add X."

**Ground first, only what exists:** mission.md, product.md, roadmap.md, tech-stack.md, CLAUDE.md, README.md, and `ls ~/.claude/skills/` (feeds Lens 2). None exist → run ungrounded and tell the user once that Lens 2 (overlap) and Lens 3 (consistency) are weaker. If the argument is empty/vague, ask one `AskUserQuestion` for the idea; if it's a multi-paragraph brief, read it as-is. Restate the idea in one line first so a misread surfaces fast, then run all seven lenses and output:

```
**[Idea restated in one line] — Verdict: [Drop it | Do it but smaller | Proceed]**
Why: [one paragraph — the 1–2 lenses that drove it, plain language, no jargon]

**Lens-by-lens:**
- **[Lens name] — [severity]:** [2–4 plain sentences. Name the exact existing feature/skill on Lens 2; cite the exact mission/roadmap/CLAUDE line on Lens 3; give rough build-estimate + use-frequency on Lens 5. If clean: "didn't fire — [one-line reason]."]
  (…all seven lenses…)

**If you build it anyway:** [2–3 sentences — which lens's prediction bites first, what the failure looks like, what to revisit]
```

**Verdict mapping (closed):** any **Critical** → "Drop it" · two or more **High** → "Do it but smaller" (name the smaller version explicitly) · mixed Medium/Low/Clean → "Proceed — it's actually fine" (+ what to watch) · all **Clean** → "Proceed — every lens cleared."

**Tone:** sycophancy is the failure mode, not bluntness — name specific failures, challenge mechanism-as-problem framing, don't insult the user (the idea is bad, the user isn't), don't soften a Critical, don't add a "great instinct!" closer, end on the verdict. If the user counters a finding, engage the specific counter and update the verdict if it changes the math — don't capitulate, don't dig in.

---

## Ground rules

- Two callers only — /build on a new project (shaping) or the user directly (verdict); never run this on your own ideas during another task.
- Plain language throughout; the user is non-technical (`voice.md`). No paths, endpoints, or stack names in user-facing text.
- Shaping loops on Reshape until Proceed or Drop; it vetoes a project only on a fatal lens. It writes no `.build-state.json` and builds no foundation.
