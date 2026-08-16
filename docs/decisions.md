# Decisions — claude-build stack rewrite

Settled choices for the structure + ASD-STE100 pass. Check this before you re-ask a question.

## 2026-08-16 — Every `schemas/` and `references/` file gets a Conciseness line.

Schema and reference docs are templates and rulebooks; §6's ASD-STE100 pass covered `SKILL.md` prose only, and left the entries agents write into these files ungoverned. Each of the 14 files now opens with `> **Conciseness.** ASD-STE100 English. Max 20 words per sentence. Max 2 sentences per entry.`, right after the title (after frontmatter, in `requirements.md`). It governs both the file's own prose and every entry an agent later writes into it. `docs/skill-standard.md` §6 and §7 record the rule.

`docs/stack-map.md` was already stale before this change — `claude-md.md`'s recorded section table (2501c, four sections) does not match the file on disk (543c, one section) as of `cff00f3`. Its generator, `scratchpad/genmap.py`, is not checked into this repo, so Part 1/2 byte counts could not be regenerated here. Run the generator and commit the refresh separately.

## 2026-08-08 — The app-shell spec is deleted. The stack does not prescribe a shell.

`references/app-shell-spec.md` (9634c) specified one generic SaaS shell in literal values — a 256px sidebar, Cmd+B to collapse, a 56px bottom nav, Sonner toasts bottom-right for 3–5s, a fixed settings category list. It did not help in practice.

The failure was not the catalogue alone. `validation.md`'s Phase 0 block restated those prescriptions as pass/fail checks, so a product that made a different and perfectly good choice got graded as failing. Prescription plus a checklist that enforces it is worse than either.

**Now:** `requirements.md ## App Shell` asks the phase to *state* its navigation, auth, settings, and universal-pattern decisions, with no catalogue to copy. `validation.md` writes one check per decision the spec actually states, and explicitly never a check for a pattern the app did not choose. The shell still exists as Phase 0 scope in `_shared/roadmap-axis.md`; what is gone is the claim that every product needs the same one.

## 2026-08-08 — Skills lose the `build-` prefix. build-lite is retired.

**The plugin qualifies the skill, so the prefix was redundant.** Invoking design meant typing `/build:build-design`. Every sub-skill directory and `name:` field drops the prefix: `spec`, `design`, `backend`, `review`, `shape`, `polish`, `migrate`. Invocation is now `/build:design`.

**The orchestrator keeps the name `build`** (`/build`). Its directory also holds `_shared/` and `schemas/`, so 46 of the stack's 50 internal paths stay untouched. Renaming it would have rewritten all of them for no user-facing gain.

**One reference form everywhere: `/build:<name>`.** It matches what the user types and the Skill tool's `plugin:skill` convention, and it stays unambiguous where the bare name would not — "design", "review", and "spec" are ordinary words in these docs.

**`/build:migrate` sweeps this rename.** Its table gained the old-to-new mapping, and its detection ladder no longer stops flat on a current-schema project: it greps for a `build-` prefixed token and runs the reference sweep when it finds one. Without that, every existing project's docs would keep naming skills that no longer answer. The legacy v1 tokens (`/build-ba`, `/build-frontend`, …) stay in the table as sweep *sources* — never rename those.

**`claude-build-lite` is retired.** The GitHub repo is archived (read-only, recoverable); the local clone is deleted. `stack: "build-lite"` is gone as a state value, and the drilling sync script no longer writes a second destination.

## 2026-08-08 — Supporting docs: two findings, and what changed.

**`drilling-discipline.md` was stale, not big.** The repo held 14056c; the canonical source at `~/.claude/skills/grill-me/references/drilling-discipline.md` held 6449c, and the header hash did not match. The file declares itself do-not-hand-edit, so the divergence was the defect. Running `scripts/sync-drilling-discipline.sh` adopted the source. The new source drops the published-Artifact interview mechanism and standardizes on `AskUserQuestion` batches; no SKILL.md depended on the artifact mechanism, so nothing broke. **Never hand-edit this file** — change the grill-me source and re-sync.

**`ux-rules.md` was never loaded and is deleted.** 11364c that no step in any skill ever read — it appeared only in /build:design's References list and as a cross-link. Its ungated rules moved to the docs an agent actually loads: platform nav, route architecture, onboarding, settings defaults, affordances, breakpoints, and motion → `app-shell-spec.md`; stable-URL, the full WCAG threshold set, and the concrete focus and keyboard standards → `gates.md`; the token discipline → a new `references/design-tokens.md`. The `N#`/`F#`/`A#` namespace is gone with it. Two wrong tags died with it: `app-shell-spec.md` tagged F21–F23 with `G-DESTRUCT` (which covers F24–F28), and cited an `N5` that never existed.

**`design-tokens.md` is new because the gap was real.** `G-TOKENS` in `gates.md` is the *checker*, and gates.md loads on external mode only — so `claude-code` had no token standard at all. The new file loads at both modes' close-out, where `design-tokens.css` is written.

## 2026-08-08 — The design references are NOT duplicates. Do not merge them.

`gates.md` loads only on /build:design external Step 3. `design-tokens.md` loads at both design modes' close-out. (`app-shell-spec.md` was the third case until it was deleted — see the 2026-08-08 entry above.) **No two are ever read by the same consumer.** Where all three need the five system states or the 44px touch target, each states it in full. Collapsing that to one owner would leave a reader with a citation it never follows. This reverses the merge that the one-home-per-rule pass would otherwise have made.

The rule holds generally: a repeated sentence is drift only when one reader loads both copies.

## 2026-08-07 (later, supersedes the entry below) — Delete the seven lenses, the deploy milestone, and demand validation.

The earlier entry called the seven-lens gate essential. The user reversed that. The reversal and its reason are the record:

- **`/build:shape` loses `## The seven lenses`, `### Step 1.3 — Bad idea check`, and `## Mode: verdict`.** They are ceremony — a gate that exists to be passed. `/build:shape` is now a shaping and research skill only: concept interview, then 3C research. The `shape-complete` write moves to the end of Step 1.2, which was the one thing that would otherwise break the resume ladder.
- **`build-deploy` is deleted whole.** Its one unique capability — the unfenced whole-app dogfood — folds into `/build:review`. On the last roadmap phase the orchestrator passes `final-phase`, and the dogfood walk drops its scope fence. That is now the only cross-phase integration check in the stack.
- **Demand validation is dropped.** Nothing in the stack now pushes back on an idea. This is deliberate, not an oversight.

`roadmap-complete` is the declared terminal state, meaning all phases are built.

## 2026-08-07 — Exploration stays. The gap is product visibility mid-pipeline. *(partly superseded)*

The first diagnosis was wrong. The felt "lock-in" is **not** the front gate. /build:shape's concept interview and 3C research are essential and stay. The constitution gate stays. *(The seven-lens gate was also called essential here; the entry above reverses that.)*

The real gap: the user loses the mental model of the final product **while moving through phases**. Evidence — `product.md` holds the End-State Vision, Screen Inventory with per-screen status, App Map, and Named Flows, and `/build:spec` replan refreshes it every phase. It is read by the orchestrator, /build:design, /build:review, and /build:shape. It is never rendered to the user. The `/eli` wrap at each phase-complete gate is phase-scoped only.

**Fix:** surface the whole-product picture at each phase boundary gate — what is built, where you are, what is left. Do not remove process.

**Open detail:** at the gate, `product.md` is still the previous phase's, because replan runs after the user picks Proceed. Either move the product.md patch earlier, or show the map plus this phase's delta.

## 2026-08-07 — Three axes, and the order they run in.

The rewrite has three separate problems. Order matters; each pass makes the next cheaper.

1. **Process** — the visibility gap above, plus the ceremony deletions.
2. **Structure** — no shared spine across the SKILL.md files; three names for a branch (`Mode:` / `Track:` / `Case`), three for a sequence (`Stage` / `Step` / `Round`); four canon rules cited *and* copied inline (~4900c of drift liability); three-way contract overlap (~3000c); two orphan schemas.
3. **Prose** — not ASD-STE100, no char budgets.

Never run the prose pass first. It tightens text that the structure pass deletes.

## 2026-08-07 — Working protocol for the rewrite.

**Order: pipeline order** — shape → spec → design → backend → review, then the off-pipeline skills, then a conformance re-check of `build/SKILL.md`.

**Per file, do all three axes at once** (standardize spine → delete duplication → ASD-STE100 to budget). Do not split them into three sweeps.

**Before each file:** re-read the stack rewritten so far, plus the next skill in the pipeline. This keeps the spine and vocabulary consistent and catches a cut that breaks a neighbour.

**Budgets:** frontmatter `description` ≤300 chars, body to a named budget, both verified with `wc -c`. Never eyeball.

## 2026-08-07 — Keep these files separate. Reason: load path, not sentiment.

A merge is wrong when it forces a reader to load text it never runs.

- `entry-point.md` + `auto-continue.md` — different axes, disjoint citation sites, zero shared sentences.
- `requirements.md` / `plan.md` / `validation.md` — three readers at three steps.
- `design-brief-external.md` + `handover.md` — written on opposite sides of the user's absence.
- `brain.md` — optional and presence-guarded, with no merge target.

Four merges did land, each for a stated reason: `pivot-protocol.md` deleted (invented nothing), the three constitution schemas merged (closed two orphans), the two drafter briefs merged (one shared envelope), and `living-docs.md` trimmed (it broke its own stated rule).

## 2026-08-07 — Measured, so nobody re-argues it.

Final: **8 SKILL.md files, 156790c → 97851c (38%)**; **23 supporting docs, 123901c → 94705c (24%)**. Total stack **288138c → 182665c across 30 files, a 37% cut**, plus `build-deploy`'s 8196c and `ux-rules.md`'s 11364c deleted whole. Not 75%. The remaining bulk is distinct mechanism. `/build:spec`, `/build:review`, `constitution.md` (87% template), `claude-md.md` (93% template), and `validation.md` (87% checklist) are all at a fidelity floor — a further cut there deletes a field or a rule, not prose.
