---
name: spec
description: >
  Translates BA session output into structured SDD documents. Three modes: (1) new project — writes mission.md + tech-stack.md + roadmap.md and scaffolds living docs; (2) start phase — creates specs/YYYY-MM-DD-[feature]/ with requirements.md + plan.md + validation.md; (3) between phases — updates living docs, runs changelog, merges branch. Always invoked by /ba or /build after a drilling session. Never standalone.
---

# /spec — Structured Document Writing

Input comes from a `/ba` session. Output is structured markdown files. No drilling, no user questions — that belongs to `/ba`.

## Invocation contract

| Mode | Model | Mechanism | Inputs (caller passes) | Outputs (skill produces) | Terminal state (this skill writes) |
|---|---|---|---|---|---|
| 1 (new project) | Sonnet | subagent | BA Mode 1 decisions verbatim | mission.md, product.md, tech-stack.md, roadmap.md + scaffolded living docs | `constitution-complete` |
| 2 (phase spec) | Sonnet | subagent | BA Mode 2 decisions verbatim, phase number, feature slug | requirements.md, plan.md, validation.md under `specs/YYYY-MM-DD-<slug>/`; feature branch created | `spec-complete` (also writes `requirementsHash`) |
| 3 (between phases) | Sonnet | subagent | BA Mode 3 replan notes verbatim, project state summary | updated WIKI.md, README.md, docs/api.md, docs/decisions.md, docs/architecture.md, CHANGELOG.md; merged feature branch | none — `/build` owns the next-phase write; this skill only nulls `currentSubStep` on clean exit |

## Mode detection

| Context | Mode |
|---------|------|
| No `mission.md` in project root | Mode 1 — New project |
| `mission.md` exists, starting a phase | Mode 2 — Phase spec |
| Phase just completed | Mode 3 — Between phases |

---

## Wiki integration

Apply `${CLAUDE_PLUGIN_ROOT}/skills/_shared/wiki.md` with `$AGENT=spec` and `$TAGS` from `tech-stack.md` (Mode 1 has no stack — pass empty).

**Friction trigger:** the user modifies a spec after approval (scope creep signal). Title: `Phase <N> spec friction: <area>`. Body: which section changed, what the delta was, why the original spec missed it. Do not repeat for the same area within one session.

**Phase-wrap trigger:** at the end of Mode 3, after living docs are updated and before branch merge. One entry per phase. Title: `Phase <N> spec: <one-line summary>`. Body: what was non-obvious about this spec, what worked well, what should be sharper next time.

---

## Latent-decisions surface (shared procedure — both Mode 1 and Mode 2)

Every spec write involves choices the user did not explicitly hand to you. Some come from BA Mode 1/2 verbatim — those are *locked*, not latent. Latent decisions are the ones you (the spec subagent) made silently while filling the templates because the BA pack didn't speak to them. They include things like: a tech-stack micro-choice (which auth library), a scoping inference (single-mart per pivot, when the BA pack didn't say either way), a structural pattern (static registry vs DB-backed), an interaction default (auto-save debounce timing), an exclusion you added to keep the phase tight, an assumption about an unspecified flow.

These are the most common source of post-approval rework — the user reads the spec, says "looks good," then discovers two phases later that something they would have argued about was decided silently. Surface them at the gate so the user can confirm or push back **before** the spec hardens.

**Procedure:** after writing the files but before the AskUserQuestion approval gate, scan your own working context for choices you made that weren't in the BA pack. Output them in this exact format as part of the same approval message:

```markdown
### Subagent design decisions you should sanity-check

These weren't in the locked BA pack — I decided them while writing the spec. None violate locked scope, but they're load-bearing.

#: 1
**Decision:** [one-sentence statement of what you decided — concrete, not abstract]
**Why I'm flagging:** [one-to-two sentences: what the alternative was, what it locks in, why a non-developer reader could miss this]
────────────────────────────────────────
#: 2
**Decision:** [...]
**Why I'm flagging:** [...]
────────────────────────────────────────
(continue numbering for each latent decision)
```

**What qualifies (include):**
- A scope inference not in the BA pack ("single-mart per pivot", "phase locks to read-only").
- A tech micro-choice not in `tech-stack.md` already ("static TS registry vs DB-introspection", "Zod for validation, not Yup").
- A behavior default the user never named ("config auto-saves on drag with 800ms debounce", "polling at 5s").
- An exclusion you added to a "Not in this phase" list because the BA pack was ambiguous.
- An assumption about an unspecified user flow ("if the user dismisses the modal, treat it as cancel, not save").
- A name or label you picked when the BA pack used a placeholder.

**What does NOT qualify (skip):**
- Anything that's verbatim from the BA decisions handoff — that's locked, not latent.
- File names, directory structure, internal variable names — implementation details, not user-facing decisions.
- Things the user has explicitly seen in mission.md / tech-stack.md / roadmap.md from a previous mode.
- Trivial defaults that have one obvious answer (CSS reset, charset declaration).

**Calibration:**
- Aim for 2–6 items per spec write. Zero items means you didn't look hard enough — every spec has at least one inference. More than 8 items means you're listing implementation noise — re-filter.
- Order by load-bearing-ness: the inference that locks the most downstream work goes first.
- Keep "Decision" to one sentence. Keep "Why I'm flagging" to one or two. The user is non-technical — name the trade-off in business terms ("means adding new fields = code change, not data change") not implementation terms ("requires a redeploy").
- If you flagged a decision in a previous round of this same spec write and the user already responded to it, drop it from the next round — don't re-litigate.

**Where it appears in the message:** after the existing self-check (story↔task↔validation coverage) and before the "Ready to proceed?" AskUserQuestion. Same chat message — the user reads coverage gaps and latent decisions side-by-side, then decides whether to approve.

---

## Mode 1 — New Project Setup

**Step 0:** Run the Read wiki procedure from the Wiki integration section. Mode 1 has no `tech-stack.md` yet — invoke with `--agent spec` and no tags.

Input: constitution decisions from `/ba` Mode 1 (mission, tech stack, roadmap, exclusions).

**Approval flow:** Write all constitution files to disk first (Steps 1–2). Then run the **Latent-decisions surface** procedure (see shared section below) — the user almost never spots silently-inferred decisions inside finished docs, so list them explicitly before the gate. Then tell the user where the files are and ask them to review directly — they will either comment in the files or in chat. Use `AskUserQuestion` only after they've had a chance to read: "Reviewed the constitution files. Ready to proceed to Phase 1 spec?" with options "Yes, proceed" / "I have changes." Do not proceed to Phase 1 until confirmed.

### Step 1 — Write constitution (4 files)

**`mission.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/mission.md`. Every field filled. Purpose: one crisp sentence. Target users: named and specific. Vision: one paragraph. Success: observable and specific.

**`product.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/product.md`. Every field filled. End-State Vision: one paragraph on what the finished product feels like to use. Screen Inventory: every screen in the finished product with phase label. Navigation Structure: flat map of how screens connect. Core Feature Surface: concrete actions a user can do at end-state. Named Flows: all flows from `/ba` Part C with phase labels on each step (3–6 flows). Phase 1 Skeleton Scope: every screen listed as live or stubbed — this is the instruction to `/frontend` to design the full IA in Phase 1.

**`tech-stack.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/tech-stack.md`. Every choice filled in. Constraints and non-negotiables listed explicitly (e.g. "strict TypeScript from commit 1", "all dependencies pinned exactly — no ^ or ~"). Explicit exclusions named with reasoning. Technical decisions table seeded with choices already made.

**`roadmap.md`** — Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/roadmap.md`. Phases numbered and sequenced. Each phase: feature name + what it delivers + why it comes at that position. Global out-of-scope named explicitly.

### Step 2 — Scaffold living docs

Create these files with headers only (no placeholder content — an empty section with a header is fine, a section with fake content is not).

**Living-doc prefix rule:** every agent-facing living doc (`WIKI.md`, `docs/architecture.md`, `docs/api.md`, `docs/decisions.md`) starts with this exact line as its first line, before the title:

```
> Agent context — not for human reading.
```

The operator visually skips files that lead with this blockquote; agents still read them. `README.md` is the *only* doc here that targets humans, so it does NOT get the prefix.

- `README.md` — project name, mission (from mission.md), setup: "TBD", status: "Phase 0 — Constitution complete. Phase 1 not started." **No prefix.**
- `WIKI.md` — prefix line, then header: "# Project WIKI", subheader: "## Tech Stack Notes", body: "*(Added per phase)*"
- `CHANGELOG.md` — prefix line, then header: "# Changelog", body: "*(Auto-generated — do not edit manually)*"
- `docs/architecture.md` — prefix line, then tech stack section copied from tech-stack.md; rest TBD
- `docs/api.md` — prefix line, then header only: "# API Surface — *(Updated per phase)*"
- `docs/decisions.md` — prefix line, then seed with decisions already recorded in tech-stack.md Key Technical Decisions table
- `polish.md` — prefix line, then headers from the polish template at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/polish.md` (Open buckets + Done). Empty body under each header. Runs in parallel with `roadmap.md` for small unscheduled work.
- `tools.md` — see Step 2a below (seeded from the user's global `~/.claude/tools.md`, not a blank scaffold).
- `handoff.md` — prefix line, then header: "# Project handoff notes" + the one-line orienting paragraph from the handoff template at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/handoff.md`. No phase sections yet — `/spec` Mode 3 appends the first one when Phase 1 completes.

### Step 2a — Seed `tools.md` from global master

Read the user's global `~/.claude/tools.md` if it exists.

- **If present:** copy the `## Machines`, `## Accounts`, and `## Services` sections verbatim into the project's `tools.md`, then append the empty `## Project-specific` section from the schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/tools.md`. Add the prefix line and a one-line header note: "Synced from `~/.claude/tools.md` at constitution time — refresh manually if global changes."
- **If missing:** write only the `## Project-specific` section (no machine/account snapshot) and surface a one-line note to the user: "Global `~/.claude/tools.md` not found — machine/account sections skipped for this project. Create the global file once to seed future projects." Do not block constitution completion on this.

### Step 3 — Seed WIKI from global WIKI

If `~/.claude/wiki/` exists: search for entries tagged with the tech stack languages and frameworks. Copy relevant entries under `## From Global WIKI — [tech]` in the project WIKI.md.

If no global WIKI yet: skip silently.

### Output

**Checkpoint state file (compaction-safe handoff):** before output, write `.build-state.json` in the project root with:
- `phase: 1`
- `feature` set to the kebab-case slug of Phase 1 from `roadmap.md`
- `step: "constitution-complete"`
- `reviewIteration: 0`
- `requirementsHash: ""`
- `currentSubStep: null`
- `dogfoodPid: null`

This is the resume anchor for the new-project path. Constitution is a phase boundary — `/build` stops here and waits for the user to re-invoke. Without the state write, a compaction during the user's pause leaves the project looking unconstituted to the next `/build`, which restarts at Mode 1.

> **Constitution written.**
> **Files created:** mission.md, product.md, tech-stack.md, roadmap.md, polish.md, tools.md, handoff.md, README.md, WIKI.md, CHANGELOG.md, docs/architecture.md, docs/api.md, docs/decisions.md
> **Phase 1 feature:** [feature name from roadmap]
> **Ready to scope Phase 1.**

---

## Mode 2 — Phase Spec

**Step 0:** Run the Read wiki procedure from the Wiki integration section — pull tags from `tech-stack.md`, invoke with `--agent spec`. Summary goes into the skill's working context.

**Friction hook:** if during this phase the user returns with spec modifications after having approved this spec, invoke the Write learning procedure with `--trigger friction --phase <N>` before re-entering Mode 2. Do not repeat for the same area within one session.

Input: phase decisions from `/ba` Mode 2 (scope, user stories, screen inventory, competitor patterns, API needs, constraints).

**Scope challenge (before writing anything):** Search the existing codebase before speccing new work.

1. For each feature in the BA handoff: does any existing file, route, component, or utility already handle part of it? List what exists and what's genuinely new.
2. Identify the minimum set of changes needed — flag if more than 8 files or 2+ new abstractions are involved and ask whether to reduce scope before writing the spec.
3. Note any deferred items in `roadmap.md` that overlap with this phase — can any be bundled without expanding scope?
4. Read project-root `polish.md` (if present). For any `## Open — *` item that touches the same surface as the current phase, flag it to the user once before writing — "polish item X overlaps with this phase, fold it in or leave it open?" Do not silently sweep polish items into the spec.

Output a brief "What already exists / What we're actually adding" summary at the top of the working context before writing. This is for the spec author only — not a user-facing deliverable.

**Approval flow:** Write all three spec files to disk first (Steps 1–3). Before presenting for approval, run this self-check and surface any gaps in one message:
- Every user story maps to at least one task group in `plan.md`
- Every task group has a corresponding check in `validation.md`
- All API contracts specify request shape, success response, and all error conditions (not just "500: unexpected error")
- Primary flow stories from `/ba` are explicitly covered in `validation.md` manual checks

Then run the **Latent-decisions surface** procedure (see shared section below) and present its output to the user as part of the same approval message. Then ask the user to review directly. Use `AskUserQuestion` after they've reviewed: "Reviewed the spec files. Ready to proceed to frontend?" with options "Yes, proceed" / "I have changes." Do not proceed to frontend until confirmed. This gate is mandatory — SDD does not allow implementation to start from an unapproved spec.

**Hash on approval (drift detector):** the moment the user replies "Yes, proceed":

1. Compute `sha256sum specs/YYYY-MM-DD-[feature-slug]/requirements.md | cut -d' ' -f1`.

2. Write `.build-state.json` with:
   - `requirementsHash` set to the computed value
   - `step` set to `"spec-complete"`
   - `phase` from `roadmap.md` (the phase being specced)
   - `feature` set to the kebab-case feature slug
   - `reviewIteration` reset to 0 (fresh phase, fresh review counter)
   - `currentSubStep` nulled
   - `dogfoodPid` preserved if present

This is the resume anchor + drift detector in one write. If context is compacted between this skill ending and `/frontend` starting, the next `/build` reads `spec-complete` and jumps straight to Step 2 (Frontend) instead of re-running `/ba` Mode 2 + the 3-doc spec write.

The hash is taken **after** the approval reply, never before — an in-progress edit must not generate a stale hash. Only `/spec` Mode 2 writes `requirementsHash`.

### Drift detection (downstream skills should run on entry)

`/frontend`, `/backend`, and `/review` recompute the hash on entry and compare to the recorded `requirementsHash`. If different:

- Surface to the user once: "requirements.md changed since spec approval. The downstream work is now building against a different contract. Continue, or restart spec?"
- Do not auto-block — the user may have intentionally tweaked a typo. Just make the drift loud.
- If user picks "Continue": clear `requirementsHash` (set to empty string) so the same warning doesn't fire repeatedly.

### Step 1 — Determine spec directory

Use today's date: `date +%Y-%m-%d`. Feature slug from the roadmap entry, kebab-case (e.g. `user-auth`, `product-listing`). Create: `specs/YYYY-MM-DD-[feature-slug]/`.

Also create a feature branch: `git checkout -b phase-N-[feature-slug]` where N is the phase number from roadmap.md.

### Step 2 — Write requirements.md

Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/requirements.md`. Fill every section:

- **Frontmatter block** (required, at top): `phase`, `type` (`initial` | `feature` | `rebuild`). These come from `/ba` Mode 2 Part A. Missing or wrong `type` causes downstream skills to misbehave (e.g. a `rebuild` with `type: feature` will preserve old UI patterns that should be deleted).
- **Scope:** one paragraph — what this phase delivers, what a user can do on completion that they couldn't before
- **User stories:** one per major user action, specific ("I can filter the list by date" not "I can manage content"). Each story must reference its Named Flow and step from `product.md` — format: `[Flow name, Step N]`. A story with no flow reference is incomplete.
- **UI requirements:** every screen + every state (default, empty, loading, error, mobile) — derived from the screen inventory collected in `/ba`. Describe **behavior and key elements**, not visual treatment. Visual treatment lives in the design file (produced by `/frontend`).
- **Data model:** all tables/schemas, field names, types, relationships
- **API contracts:** every endpoint needed by the frontend. Method, path, auth, request shape, success response, all error conditions. Frontend and backend both build from these — they are the shared contract.
- **Constraints & context:** business rules, patterns to follow from tech-stack.md, tone
- **Excluded from this phase:** named explicitly

Validation rule: every API contract must have error responses specified. "500: unexpected error" alone is not sufficient — name the domain-specific error conditions.

### Step 3 — Write plan.md

Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/plan.md`. Organize as numbered task groups, independently reviewable. Each group = one coherent slice that code-harness can implement and verify before moving on.

**Design-agnostic contract.** `plan.md` is written *before* `/frontend` produces the design file. Any visual specifics written here (hex values, Tailwind classes, pixel sizes, font names) will go stale once the design is approved and will silently contradict the design file. Do not include them.

- Describe **what each group delivers** (user-visible capability, data path, API surface, integration point)
- Name components, files, modules, endpoints specifically — but do not describe their visual treatment
- Each group includes a one-line **Verify:** field with the command that proves it's done
- Each group includes a **Depends on:** field listing earlier groups it requires

Typical group structure:
- Group 1–2: Shared primitives, layout shell (components named; visuals come from design file at build time)
- Group 3: Page-level composition (e.g. "rebuild the board page — cards, columns, drag-and-drop wiring")
- Group 4: Database layer (if any)
- Group 5: API routes (if any)
- Group 6: Integration, edge cases, error handling
- Group 7: Story walk — verify every user story end-to-end

Sub-tasks within groups should be specific enough to implement without ambiguity: "Create `src/routes/products.ts` with GET /api/products returning paginated list from DB" is good. "Add product route" is not. "Use `bg-[#F9FAFB]`" is **out of scope for plan.md** — that belongs in the design file.

### Step 4 — Write validation.md

Use schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/validation.md`. This is the test contract — `/review` reads this file and executes every check.

- **Automated checks:** specific commands that exit 0 on success. TypeScript typecheck, unit tests (named), API endpoint verification with curl
- **Manual verification:** specific steps at named viewport sizes. Each step is binary (pass/fail)
- **Definition of Done:** all criteria listed, all true before phase is approved

Validation rule: every user story in requirements.md must have at least one corresponding manual verification step. If a story isn't in validation.md, it won't be tested.

### Output

> **Phase [N] spec written.**
> **Directory:** `specs/YYYY-MM-DD-[feature]/`
> **Branch:** `phase-N-[feature-slug]`
> **Spec:** [N] user stories, [N] screens, [N] API contracts, [N] plan groups, [N] validation checks
> **Ready for frontend design.**

---

## Mode 3 — Between-Phase Docs Update

**Step 0:** Run the Read wiki procedure from the Wiki integration section — pull tags from `tech-stack.md`, invoke with `--agent spec`.

**Phase-wrap hook:** after Step 6 (changelog) and before Step 7 (merge branch), run the Write learning procedure with `--trigger phase-wrap --phase <N>`. Exactly one entry per phase — draw the summary from the spec files and git log of the just-completed phase.

Input: replan notes from `/ba` Mode 3. Run this in the same session the phase completed — not later.

### Step 1 — Update README.md

Change status line to: "Phase N complete — [feature]. Phase N+1 next: [feature from roadmap]."

### Step 2 — Update WIKI.md

Add a new section: `## Phase N — [Feature Name] Learnings`

Write what was learned, not what was done. Git log has the what. WIKI has the why and the surprise. Minimum 3 entries if anything non-obvious happened. Each entry: topic + what was learned (useful to a future agent starting fresh). Examples: gotchas, tech stack quirks, patterns that worked, patterns that failed.

### Step 3 — Update docs/api.md

Add or update every endpoint that changed this phase. Keep it current — this is the live API surface, not the spec (which is frozen after approval).

### Step 4 — Update docs/decisions.md

Add any non-obvious technical choices made during implementation. Format: `**[Decision]** — Why: [reason]. Alternatives: [what was considered and rejected].`

### Step 5 — Update docs/architecture.md

If new components were added or the data model changed, update the relevant sections.

### Step 5a — Append phase handoff to `handoff.md`

`handoff.md` lives at the project root and is **distinct from `specs/<dir>/handover.md`** (which is the frontend → backend frame index). See `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/handoff.md` for the full schema and write rules.

Append exactly one new section to project root `handoff.md`:

```markdown
## Phase <N> — Handoff Notes  *(written <YYYY-MM-DD> by /spec Mode 3)*

### What a fresh session might re-derive painfully
- [...]

### Operator-side context
- [...]

### Things that almost made it into permanent docs
- [...]
```

Source the bullets from:
- The just-completed phase's git log + `/review` report (for re-derive-painfully items).
- Anything the user said about how / where / on what device they dogfooded (for operator-side context).
- Anything you flagged during the phase that didn't quite warrant a `mission.md` / `tech-stack.md` / `WIKI.md` / `docs/decisions.md` / `CLAUDE.md` / `polish.md` update.

If a section has nothing this phase, write `*(No fresh-session gotchas this phase.)*` rather than leaving the section blank — the explicit empty signal matters.

Append only — do not edit prior phases' sections. Exactly one new section per phase.

### Step 5b — Migrate any polish items closed this phase

Read `polish.md`. For any item under `## Open — *` that was incidentally fixed by the phase's work, move the line to `## Done — Phase <N>` with a one-line note (e.g. "Fixed in Phase 2 — button now scales on mobile."). Do not delete items from polish.md — moving to Done preserves the user's trail of what got addressed.

If no polish items were touched, skip this step silently.

### Step 6 — Run changelog

Generate `CHANGELOG.md` by running:

```bash
git log --format="%ad|%s" --date=short | awk -F'|' '
  {
    if ($1 != date) { date=$1; print "\n## " date }
    print "- " $2
  }
' >> CHANGELOG.md
```

Or if the project has a `scripts/changelog.py`: `python3 scripts/changelog.py`

Review the output — remove merge commits and branch housekeeping. Keep only meaningful changes.

### Step 7 — Merge branch

```bash
git checkout main
git merge phase-N-[feature-slug] --no-ff -m "Phase N complete: [feature name]"
git branch -d phase-N-[feature-slug]
```

### Output

> **Docs updated and branch merged.**
> **WIKI:** [N] learnings added for Phase N
> **CHANGELOG:** updated through [date]
> **Branch:** `phase-N-[feature-slug]` merged to main
> **Next:** Phase N+1 — [feature from roadmap]

---

## Ground rules

- Write draft files to disk first (Mode 1 and Mode 2). Tell the user where to find them and ask them to review directly. Use `AskUserQuestion` to confirm after they've reviewed — not before writing. This gives the user something real to react to, not a text summary.
- Every field in every schema must be filled. Empty sections are failures — write them or remove the heading.
- API contracts must include all error conditions, not just success responses.
- Every user story must have a corresponding validation check.
- Docs update (Mode 3) happens in the same session as phase completion — not deferred.
- WIKI records what was learned, not what was done.
