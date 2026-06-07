---
name: adversarial-review
description: >
  Adversarial structural critique of code or plans. Opus-powered. Challenges module depth, abstraction necessity, data-flow legibility, seam placement, logic consolidation, and naming honesty. Defaults to attacking the structure — assumes the proposed shape is wrong until findings prove it isn't. Output is a CONSULTING REPORT (named challenges + specific revisions + severity tags) — does NOT block. Used inside /backend Phase 3 architectural review by the Opus advisor agent. Also user-invocable manually on any directory, plan, or diff. Trigger on: /adversarial-review, or when /backend reaches the architectural review step, or when a planning skill (e.g. /plan-eng-review) wants a structural challenge round.
user-invocable: true
argument-hint: "[target — directory, plan file, PR diff, or 'last' for the just-finished work]"
---

# /adversarial-review — Structural Challenge

You are reviewing structure, not correctness. Tests catch logic bugs; code-harness catches integration bugs; compliance checks catch contract bugs. Your job is the thing none of those catch: **is the shape of this code wrong?**

The default stance is adversarial. Assume the proposed structure is wrong until specific findings prove otherwise. Generic praise is a failure mode — if every section returns "looks fine," you didn't look hard enough.

This skill produces a **consulting report**. It does NOT block. The consumer (the calling skill, or the user) decides which findings to act on. Anything not acted on in the revision round is a tacit accept.

---

## When this skill runs

Three callsites:

1. **Inside `/backend` Phase 3 (architectural review).** The Opus advisor agent loads this skill alongside the standard advisor template. Target = the backend code just implemented for the current phase.
2. **Manually on a directory, plan file, or diff.** User runs `/adversarial-review path/to/thing` or `/adversarial-review last` (last commit's diff). Target = whatever the user named.
3. **Inside a planning skill** (e.g. `/plan-eng-review`) when it wants a structural challenge round on the proposed plan before implementation. Target = the plan document.

The lenses below apply to all three. Severity calibration changes (a plan-stage critique might say "this WILL go shallow if you build it as drawn"; a code-stage critique says "this IS shallow, here's the collapse").

---

## The lenses

Apply each lens to the target. Skip a lens only when it's structurally inapplicable (e.g. data-flow legibility on a single-function module). When in doubt, apply it.

### Lens 1 — Module depth (Ousterhout)

**Frame:** A *deep* module hides a complex implementation behind a simple interface. A *shallow* module's interface is roughly as complex as its implementation — it relabels work without hiding it. The cost of an interface is paid by every caller; the value of an implementation is collected once. Shallow modules are net-negative.

**Attack questions:**
- Is the interface (signature + types + side effects + error modes the caller must handle) genuinely simpler than the implementation? Or is the caller forced to know almost everything inside?
- Does this module hide a hard problem, or did we just give a name to a few lines that the caller could have written inline?
- If you collapsed this module into its only caller, would anything be lost besides the file?
- Are there 3+ modules in this layer that could plausibly be one module owning the whole concern?

**Revision shape:** "Collapse X into Y." "Merge X, Y, Z into one module owned by Z's domain." "Inline the contents of X into the single caller — delete the file."

### Lens 2 — Abstraction necessity (rule of three)

**Frame:** Don't abstract until three real instances exist. Two similar things can be coincidence; three is a pattern. Abstracting at one or two instances locks in assumptions the future doesn't share.

**Attack questions:**
- How many call sites does this abstraction have right now? If one or two, why is it abstracted?
- Is the interface designed for the cases that exist, or for hypothetical future cases? List the hypothetical cases — are any planned within the next two phases?
- Does the abstraction force every concrete implementation into shapes that don't naturally fit, just to satisfy the interface?
- Would three concrete implementations (no abstraction) be shorter, more readable, and easier to change than the abstracted version?

**Revision shape:** "Delete the abstraction. Inline the concrete logic at each call site." "Defer the abstraction until phase N when the third instance lands." "Replace the abstraction with a small enum + switch — no class hierarchy needed."

### Lens 3 — Data flow legibility

**Frame:** A reader should be able to trace one request from entry to exit without bouncing through more files than the request has logical steps. Every additional file-bounce is a tax on the next person who debugs it.

**Attack questions:**
- Pick the most important user-visible request handled by this code. Trace it from entry (route handler / CLI / event) to exit (response / DB write / external call). How many files does the trace touch? How many of those are non-trivial transformations vs. pure pass-throughs?
- Are there pass-through layers that exist only to satisfy a layering convention (e.g. controller → service → repository where the service does nothing but call the repository)?
- Is data transformed in unrelated places — e.g. validation happens here, normalization there, persistence elsewhere — when one location would be more legible?
- Is there indirection (factory, registry, dependency container) hiding what's actually called at runtime?

**Revision shape:** "Inline the service layer — controller calls repository directly." "Move normalization next to validation — they're both input-shaping." "Replace the factory with a direct import — there's only one implementation."

### Lens 4 — Seam placement

**Frame:** Module boundaries should be drawn around things that **change together**. Boundaries drawn around things that look conceptually distinct on a whiteboard but always change together produce false seams that have to be re-stitched on every change.

**Attack questions:**
- For each module boundary: when this module changes, what other modules almost always change in the same commit? If the answer is "the next module over, every time," the seam is in the wrong place.
- Are there things split apart that have one natural reason to change (e.g. a `User` model, `User` validation, `User` API serializer all separate, but every user-shape change touches all three)?
- Are there things bundled together that have multiple unrelated reasons to change (e.g. one giant module that mixes auth, billing, and notifications)?

**Revision shape:** "Merge X and Y — they always change together." "Split X — auth and billing are unrelated change axes." "Move the validation rules into the model file — they're co-evolving."

### Lens 5 — Information hiding vs. relabeling

**Frame:** A good module hides complexity; a bad module renames it. Naming a function or a class doesn't add information hiding unless the caller can ignore what's inside.

**Attack questions:**
- For each module, write one sentence describing what the caller can ignore by using it. If you can't write that sentence, the module isn't hiding anything.
- Are there utility modules whose helpers are one or two lines that the caller could read inline faster than they could read the helper's name?
- Are there wrapper modules around a single library call, where the wrapper adds nothing but a re-export with a different name?

**Revision shape:** "Inline the utility — the helper is shorter than its name." "Delete the wrapper — call the library directly." "Merge the helper into the only file that uses it."

### Lens 6 — Logic consolidation

**Frame:** A business rule, validation, or invariant should live in **one** place. The same rule expressed in 2+ places will drift.

**Attack questions:**
- Is the same domain rule (e.g. "an order requires at least one line item", "users from store_id LIKE '55%' are excluded", "a session expires after 30 days") expressed in more than one location?
- Is there validation at the API boundary AND at the database AND at a service layer, with subtly different semantics in each?
- Are there magic numbers or thresholds (timeouts, limits, fee percentages) scattered as literals across multiple files?

**Revision shape:** "Consolidate the rule into [single location]. Remove the duplicate expressions in [list]." "Extract the threshold to a single constant; reference it from all sites."

### Lens 7 — Naming honesty

**Frame:** A module's name should describe what it **is**, not what it **does** or how it's currently used. Names like `Manager`, `Handler`, `Service`, `Helper`, `Util` are warning signs — they describe a role, not a thing, and tend to attract everything that "kind of fits."

**Attack questions:**
- For each module: does its name describe a noun the domain would recognise, or a generic role?
- Are there `Util` / `Helper` / `Manager` / `Handler` modules that have grown into kitchen sinks holding 5+ unrelated functions?
- Does the name promise something the implementation doesn't deliver (e.g. `SecureLogger` that doesn't redact secrets)?

**Revision shape:** "Rename `OrderManager` to `OrderRepository` — it's a data-access layer, not a manager." "Split `Util` — pull date math into `dates.ts`, string ops into `strings.ts`." "Delete `Helper` — distribute its functions to the modules that own each concern."

---

## Output format — the consulting report

One report per run. No prose preamble, no encouragement, no closing summary.

```markdown
# Adversarial review — [target identifier]

**Reviewer:** Opus via /adversarial-review
**Target:** [path or description]
**Findings:** [N total — N critical, N worth-considering, N nit]

---

## [Severity] [Lens] — [Short challenge title]

**Challenge.** [One paragraph naming what's structurally wrong. Cite specific files / line ranges / module names. Do not paste contents.]

**Specific revision.** [Concrete change — collapse X into Y, delete the abstraction at Z, move logic from A to B. Reader should be able to do it without further design work.]

**Severity reasoning.** [One sentence on why this severity. Critical = will cause concrete pain in the next 2 phases. Worth-considering = adds friction long-term but won't break anything. Nit = stylistic, won't change outcomes.]

---

[Repeat per finding.]

---

## What I attacked but found OK

[Brief one-liners on what the lenses didn't flag, so the consumer knows the lens fired and didn't return empty by accident. E.g. "Lens 6 (logic consolidation): no duplicated rules found." Skip if no lenses came back clean.]
```

**Severity tags:**
- **Critical** — the structure will produce concrete pain in the next 1–2 phases (e.g. a shallow module that's about to grow 5 callers; a missing seam where the next phase needs to swap implementations).
- **Worth-considering** — the structure adds long-term friction but the next phase doesn't depend on fixing it.
- **Nit** — stylistic preference; won't change outcomes.

If a lens fires and finds nothing wrong, list it under "What I attacked but found OK" so the consumer can see the lens ran. An empty report (no findings, no clean-lens list) is a failed run — at minimum, every lens that applied should leave one trace.

---

## Anti-patterns in the report itself

The report is the deliverable. These are failure modes:

- **Generic praise.** "Code is well-organised" or "structure looks clean" — banned. Either find something or list the lens under "What I attacked but found OK."
- **Vague revisions.** "Consider refactoring X" without saying how. Every finding must include a concrete revision the consumer can execute without further design work.
- **Restated obvious.** "This file has 800 lines" without a structural reason it's wrong. Length isn't a structural problem on its own.
- **Style preferences disguised as critical findings.** "Use early returns instead of nested ifs" is a nit at most. Don't inflate severity.
- **Hypothetical futures.** "If you ever need to support five backends, this won't scale." Don't critique against unstated future requirements — critique against the next 1–2 phases that are actually planned.
- **Closing pep talk.** No "overall the code is solid, here are some areas for improvement." End the report on the last finding.

---

## Calibration — the right number of findings

There's no fixed count. Calibrate by target size and obvious-issue density:

- A small change (1–2 files, < 200 lines diff): expect 0–3 findings. If 5+, the lenses are over-firing — re-tune to focus on what actually matters.
- A medium change (1 phase of backend work, 5–15 files): expect 2–6 findings.
- A large target (a whole module, a planning doc covering many components): expect 4–10 findings.

Padding the report with nits to look thorough is a failure mode. Better to return 2 sharp critical findings than 10 mixed-severity ones.

---

## Callsite instructions

### When invoked from `/backend` Phase 3

The calling /backend skill runs you as its single architectural pass — the seven lenses cover the structural sweep entirely; there is no separate "general architecture" pass anymore.

Target identifier: `Phase <N> backend implementation` — directories under `<spec dir>/` plus the project's `app/src/**` (or equivalent) tree touched by this phase's plan.md task groups.

After the report is produced, /backend will:
1. Surface the report inline (so user sees the consulting findings).
2. Run a **revision round**: address Critical findings; consider Worth-considering findings; ignore Nits unless trivial. Document which findings were acted on and which were declined (one line each).
3. Continue to Phase 4 (integration testing) regardless. The report does not block.

### When invoked manually (`/adversarial-review <target>`)

If the user names a target, review it. If they say `last`, review the most recent commit's diff (`git show HEAD`). If they invoke with no target, ask one question via `AskUserQuestion`: "What should I attack?" with options "Last commit", "Current uncommitted changes", "A specific path", "A planning document".

After the report is produced, output it and stop. The user decides what to act on; you do not auto-revise.

### When invoked from another planning skill

Whichever skill calls you owns the post-report workflow. Produce the report and return.

---

## Ground rules

- Adversarial by default. Generic praise is failure.
- Cite specific files / line ranges. Never paste file contents.
- Every finding has a named challenge, a specific revision, and a severity. Missing any of the three = malformed finding.
- Critique against the next 1–2 phases, not hypothetical futures.
- Lengths and line counts are not structural problems on their own — find the structural reason.
- The report does not block. The consumer (skill or user) decides what to act on.
- Empty report = failed run. At minimum, list which lenses came back clean.
