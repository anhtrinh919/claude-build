# Changelog

All notable changes to `claude-code-sdd` are documented here.

## [0.5.0] — 2026-06-07

### Changed — outcome-driven gates (the headline)
- **You approve outcomes, never spec files.** `/ba` Mode 2 now ends with a user-approved **Outcome Card** (`outcome-card.md`, ~10 lines: phase goal, what you'll be able to do, what "it worked" looks like on screen). It is the only spec-time gate.
- **`/spec` Mode 2 auto-proceeds.** Parallel drafters (Sonnet: requirements + validation; Opus: plan) → main-session reconciliation → a **3-skeptic adversarial panel** (completeness/error-paths, testability, outcome-traceability + scope creep) → fix loop (cap 3) → auto-proceed. Experience-affecting latent decisions are asked immediately as one-off questions; invisible-technical ones go to the panel + `docs/decisions.md`.
- **Constitution gate is outcome-only.** `/build` presents a plain-language product story; `tech-stack.md` is machine-written and never shown.
- **The card flows end-to-end.** `validation.md` gains an Outcome Checks section; `/review` 2f grades every card outcome (Partial/No on a primary = HIGH) and the report leads with the per-outcome verdict; the phase-end dogfood handoff's "What you can test" bullets are generated from the card.

### Added — parallel-agent upgrades
- **Backend foundation overlap.** `plan.md` groups carry a `Design-dependent: yes/no` tag; on external design tracks, `/build` spawns a background agent that builds the `no` groups (data layer, API routes) while you design. `/backend` skips any group whose verify script already passes. New `foundationStatus` state field.
- **Blind reviewer fleet.** `/review` Pre-2a.5 runs three Engine-2 reviewers (first-timer desktop / impatient mobile / returning user) with a 2-of-3 confirmation rule; 1-of-3 flags are re-verified before entering the fix loop. Engine 2 gains a persona slot.
- **Background competitor research** in `/ba` Mode 2 — runs while you answer the scope drill.
- **`skills/_shared/subagent-policy.md`** — the stack-wide rulebook: one-level nesting, briefing styles, model aliases (no version pins), output contracts, verify-on-return, parallel-dispatch rules.

### Fixed — subagent architecture
- **One-level nesting rule enforced.** Subagents cannot spawn subagents; `/spec` Mode 2 and `/review` now run inline in the `/build` session, and `/frontend`'s Track-3 brief is written inline with `/impeccable` loaded (was a nested spawn that silently degraded).
- **`/backend` wave dispatch.** Task groups dispatch in dependency-ordered waves; agents never commit, never start dev servers, never address the user — the main session integrates, runs visual gates, and commits per group.
- Model version pins removed everywhere ("Opus 4.7" → session default; invalid pinned-ID references → aliases).

### Removed
- `polish.md` / `tools.md` / `handoff.md` scaffolds and their `/spec`/`/build` wiring (reverted upstream — the public repo now mirrors the maintained stack exactly).

## [0.4.0] — 2026-06-06

### Added
- **`/bad-idea`** — contrarian reality check skill. Two modes: (1) full lens-by-lens verdict (manual); (2) tight verdict + minimum-viable-shape (invoked by `/build` at the top of every new project). Seven lenses: solution-as-problem framing, existing-solution overlap, internal consistency, demand reality, cost/value sanity, bloat detection, one-size-fits-all. Surfaced via `AskUserQuestion` loop until the user picks Proceed / Refine / Drop.
- **`/adversarial-review`** — structural critique skill. Opus-powered. Attacks module depth, abstraction necessity, data-flow legibility, seam placement, information hiding, logic consolidation, and naming honesty. Called by `/backend` Stage 3 for architectural review; also user-invocable on any directory, plan, or diff. Produces a consulting report — does not block.
- **`/dogfood`** — browser-driven verification gate for work done outside `/build`. Feature check mode (default) and full sweep mode. Three-signal gate: functional scenarios, blind naive-reviewer goal-check, friction-free primary path. Fix loop with 3-attempt cap per failing signal. Uses `/browse` only; never `mcp__claude-in-chrome__*`.
- **`skills/_shared/wiki.md`** — shared wiki integration reference for all SDD skills (read before first step, write on friction + phase-wrap).
- **`skills/_shared/browser-review-engine.md`** — shared browser review engine used by `/dogfood` and `/review`. Four engines: scenario subagent, naive reviewer, three-signal gate, fix loop.
- **`skills/_shared/voice.md`** — shared voice rules for all SDD skills (non-developer language, AUQ for decisions, no process talk).
- **`skills/build/schemas/polish.md`** — schema for the project-root `polish.md` file (small bugs / polish / deferred requests, parallel to roadmap).
- **`skills/build/schemas/tools.md`** — schema for the project-root `tools.md` file (machines, accounts, services, project-specific env vars).
- **`skills/build/schemas/handoff.md`** — schema for the project-root `handoff.md` file (per-phase fresh-session gotchas, one section per phase).

### Changed
- **`/spec` Mode 1** — now scaffolds `polish.md`, `tools.md`, and `handoff.md` alongside the constitution. `tools.md` is seeded from the user's global `~/.claude/tools.md` if it exists (copies Machines / Accounts / Services sections, appends empty Project-specific stub).
- **`/spec` Mode 3** — now appends a per-phase section to `handoff.md` between the architecture update and changelog steps.
- **`/build` project state prime** — now reads `polish.md` (item count + 3 most recent) and `handoff.md` last section on every next-feature entry. Both surfaced in the `## Project state` working-context summary.
- **`skills/build/schemas/living-docs.md`** — updated file tree to include `product.md`, `polish.md`, `tools.md`, `handoff.md`; added description sections for all three new project-root files.

## [0.3.0] — 2026-04-28

### Added
- **`bin/tdd-config`** — sanctioned CLI wrapper for managing per-project tdd-guard config without wiping the project's accumulated `ignorePatterns`. Subcommands: `enable`, `disable`, `ignore <pattern>`, `ignore --remove <pattern>`, `show`. First `enable` in a project seeds an SDD-aware ignore list (verify scripts, migrations, schemas, generated code, build configs). `/backend` and `/code-harness` now both call it instead of writing the JSON file directly.
- **Phase-end dogfood handoff** — when `/review` clears a phase, `/build` auto-starts a long-running dev server, detects a LAN-reachable URL (Tailscale-aware), seeds env-driven dogfood credentials when the project supports them, and prints an operator-voice "what you can test" list mapped from your user stories. Skipped on `phase-blocked`.
- **`ui: true | false` flag in `requirements.md` frontmatter** — explicit signal for whether `/review` Step 1.5 (visual compliance) should run. Pure backend / infra phases now set `ui: false` and skip visual checks instead of having the gate guess from screen count.
- **`requirementsHash` drift detection** — `/spec` records a sha256 of `requirements.md` at user approval; downstream skills (`/frontend`, `/backend`, `/review`) recompute on entry and surface a warning if the contract changed after approval.
- **`reviewIteration` counter + auto-fix loop in `/review`** — HIGH and MEDIUM issues now route through an internal subagent that re-runs `/code-harness` instead of escalating technical decisions to the user. Cap is 3 iterations, then a binary surface (Accept anyway / Stop). LOW-only issues are fixed silently.
- **Phase health score** — `/review` Step 2 reports a 0–100 weighted score across functional, visual, UX feedback, mobile, edge-state, and navigation categories.
- **Shared reference at `skills/_shared/pencil-preflight.md`** — canonical Pencil MCP pre-flight (launch via Bash for both macOS and WSL2, never kill processes mid-session, plain-JSON escape hatch). `/frontend` loads it before any Pencil MCP call.

### Changed
- **`/ba` rewrite** — stops asking technical questions that have no user-facing tradeoff (auth provider, DB engine, deploy target, state library, build tooling). New "drilling discipline" section enforces decision-tree depth (5–50 questions in Mode 1, 5–20 in Mode 2, 0–5 in Mode 3) with per-mode floors. Mode 2 no longer re-scopes — it confirms and details the roadmap entry. Hands off a "primary flow" list (1–3 stories) that becomes `/review`'s stop criteria.
- **`/spec` Mode 1** — three-doc split (`mission.md` + `tech-stack.md` + `roadmap.md`) is the source of truth; the older single `constitution.md` schema is removed. Living docs (WIKI, architecture, api, decisions) get an "agent context — not for human reading" prefix so the operator visually skips them.
- **`/spec` Mode 2** — adds a scope-challenge step (search the existing codebase before speccing new work; flag if >8 files or 2+ new abstractions are involved) and a self-check on every story / task group / API contract before requesting approval.
- **`/build` state schema** — typed shape with `phase`, `feature`, `step`, `reviewIteration`, `requirementsHash`, `currentSubStep`, `dogfoodPid`. Step enum normalized: `*-complete` is the new canonical (`*-approved` legacy values still read and auto-migrated on the next gate write). New `phase-blocked` state for cap-hit `/review` outcomes — does not auto-resume, surfaces blockers and waits for the user to choose (another fix round, manual rollback, accept-and-move-on).
- **`/build` same-session auto-continue** — phase-boundary gates (`constitution-complete`, `phase-complete`) now skip the manual re-invoke when the prior gate was written in the same uncompacted session. Cold-start (new session, post-compaction) keeps the manual gate.
- **`/frontend` design-tool persistence** — the choice between external tool and Claude Code is now a project-level decision recorded in `mission.md` under `## Design Tool`. Asked once at Phase 1, never again. Plus: design-quality gate (trunk test + AI-slop check) before Phase 4 approval.
- **`/code-harness` and `/backend`** — both phases now toggle tdd-guard via the bundled `tdd-config` instead of writing the JSON file. Explicit warning never to hand-write `config.json`.
- **`/backend` Phase 3 architectural review** — phrased as "spawn a subagent for architectural review" instead of pinning to a specific advisor template path; runs Opus where available.
- **`/review` Step 0** — new "scope drift detection" sweep before automated checks: classify every plan.md task group as DONE / PARTIAL / NOT DONE / SCOPE CREEP from `git diff`. Primary-flow groups in NOT DONE / PARTIAL trigger the auto-fix loop instead of running Step 1.

### Removed
- `skills/build/schemas/constitution.md` and `skills/build/schemas/phase-spec.md` — superseded by the `mission.md` + `tech-stack.md` + `roadmap.md` Mode 1 split and the `requirements.md` + `plan.md` + `validation.md` Mode 2 split.
- Companion-plugin link in README — wiki has been bundled inside this plugin since v0.2.0; the standalone `claude-sdd-wiki` plugin is fully retired.

## [0.2.0] — 2026-04-25

### Changed
- **Bundled the wiki memory CLI inside this plugin.** The companion plugin `claude-sdd-wiki` is now archived; its CLI ships at `scripts/wiki.mjs` and is invoked by skills via `node "${CLAUDE_PLUGIN_ROOT}/scripts/wiki.mjs"`. Single-plugin install — no companion to track.
- Skills now require `node` on PATH (already de-facto required by Claude Code itself). `bun` requirement is unchanged (verify harness only).
- `/preflight` updated: drops `claude-sdd-wiki` row, adds `node` row.
- README, `docs/QUICKSTART.md`, `docs/INSTALL-DEPS.md` rewritten so first-time readers see one cohesive product. No "companion plugin" framing.

### Migration for v0.1.0 users
- Existing wiki entries at `~/.claude/wiki/` are unaffected — same data dir.
- Optionally uninstall the standalone `claude-sdd-wiki` plugin once you're on v0.2.0 of this plugin: `/plugin uninstall claude-sdd-wiki`.

## [0.1.0] — 2026-04-25

Initial public release.

### Added
- Seven skills bundled as a Claude Code plugin:
  - `/build` — master orchestrator with crash-safe state file
  - `/ba` — business-analyst drill (3 modes)
  - `/spec` — structured doc writer (3 modes)
  - `/frontend` — design brief + handover
  - `/backend` — implementation with visual compliance gate
  - `/review` — spec compliance + UX dogfooding
  - `/code-harness` — per-task discipline pipeline
  - `/preflight` — first-run dependency check
- Phase-type awareness (`initial` / `feature` / `rebuild`) in `requirements.md` frontmatter so rebuild phases override existing UI patterns instead of preserving them.
- Visual compliance gate inside `/backend` and `/review` — screenshot the built UI, compare against the design frame, fix drift before commit.
- Design-tokens pipeline — `/frontend` extracts variables from the design file into `design-tokens.css`; `/backend` imports it.
- Optional graceful integration with `claude-sdd-wiki` for per-agent memory across sessions; pipeline degrades silently when not installed.
