# Living Docs — Structure & Update Rules

Living docs record project state, each at its own level of permanence (see Update cadence below). Agents read them before touching code. Never scan raw source files when a living doc covers the same ground.

## File Structure

Every SDD project contains:

    project/
    ├── CLAUDE.md
    ├── mission.md
    ├── product.md
    ├── tech-stack.md
    ├── roadmap.md
    ├── backlog.md
    ├── README.md
    ├── CHANGELOG.md
    ├── specs/
    │   └── YYYY-MM-DD-[feature]/
    │       ├── outcome-card.md
    │       ├── requirements.md
    │       ├── plan.md
    │       ├── validation.md
    │       ├── design-brief.md
    │       ├── design-tokens.css
    │       ├── mockups/
    │       └── handover.md
    └── docs/
        ├── architecture.md
        ├── api.md
        ├── decisions.md
        └── deployment.md

Each file's purpose and authoring detail lives in its own schema under `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/` (or, for `docs/deployment.md`, in `build-deploy/SKILL.md`) — this doc governs update cadence and cross-doc rules, not per-file content.

## Update cadence

**Always current — refreshed every phase wrap:** `product.md`, `tech-stack.md`, `roadmap.md`, `docs/api.md`, `docs/decisions.md`, `CLAUDE.md`, `README.md`, `CHANGELOG.md`.

**User-approved, frozen:** `outcome-card.md` — the only doc actually surfaced to the user verbatim and approved via `AskUserQuestion`. A later change restarts build-spec phase mode.

**Agent-authored snapshot — Claude's own draft, not user-approved; update without ceremony as reality diverges:** `mission.md` (a user-*felt* change still routes through the standard felt-impact-fork rule — no heavier pivot required beyond that), `requirements.md` / `plan.md` / `validation.md` (each phase's own dated copy; `requirements.md` keeps its within-phase sha256 drift-hash — a design/backend execution contract, not a user-approval claim), `design-brief.md`, `handover.md`, `design-comment.md`.

**Conditional:** `GLOSSARY.md` — same felt-need trigger as `design-brief.md`'s claude-code track, when vocabulary is genuinely ambiguous. (`INTERACTIONS.md` evaluated — deliberate skip, narrower fit than this stack needs.)

## File Descriptions

### CLAUDE.md
The auto-loaded agent operating brief (Claude Code reads it at the start of every session in the project dir). It is an **index + operating context**, not a content doc: a one-line product description, how to resume a build, pointers to every living doc, and a `## Project directives` section. It never duplicates the docs it points to — on conflict, the pointed-to doc wins. The `## Project directives` section is the home for durable, project-specific user instructions/preferences/constraints stated in conversation that fit no other doc; agents append a dated one-liner the moment one surfaces. Created once by `/build-spec` (constitution mode), refreshed each phase by `/build-spec` (replan mode). Schema: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/claude-md.md`.

### product.md
Screen inventory, navigation, named flows, and the App Map — **as actually built, not the original plan.** Maintained per phase by `/build-spec` (replan mode): new screens added, cut screens marked `removed`, reshaped screens marked `changed`, App Map regenerated. This is the phase-start re-board's drift anchor, so it must reflect reality. Schema: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/product.md`.

### backlog.md
The short-term task lake — transient work that doesn't belong in any phase's spec. Two zones: **Reports** (threaded dogfood bugs/feedback, `DF-N`, with rounds and status — the chat-driven replacement for an external bug tracker) and the flat task buckets (`T-N`: roll-in candidates, dogfood polish, side tasks). The agent maintains it from conversation and always tells the user the item ID. NOT the roadmap, NOT directives. Schema: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/backlog.md`.

### README.md
Project name, mission (one sentence from constitution), setup instructions (how to run locally), current status line: "Phase N — [feature name] — in progress / complete." Update when a phase completes or setup steps change.

### docs/architecture.md
Component map (what exists and what it does), data models (schema with fields and types), tech decisions (link to docs/decisions.md for the why). Update when a new component is added or the data model changes.

### docs/api.md
Current full API surface. Every endpoint: method, path, auth required, brief description of what it does. Update any time an endpoint is added, changed, or removed. This is the live truth — the spec file is only a phase's snapshot.

### docs/decisions.md
The project's **decision ledger** — the within-project analog of the brain (which is cross-project canon). **Re-read it before reopening any settled question, at phase start, and on every post-compaction reload**, so a new session or compacted agent can't silently overturn what's already decided. Two sections:

**## User decisions (settled — do not reopen without asking).** Every fork the user resolved. This is the anti-overturn record: a new or post-compaction agent reads it and honors it instead of re-litigating. One entry per decision — the question, the options offered, **what the user chose**, the one-line *why* (if stated), and the phase/date. An agent that believes a settled decision should now change must surface the prior choice to the user ("you chose X for this because Y — reopen?") — never quietly pick differently. Append-only; never delete a settled decision (mark it superseded if the user reopens it). Format:

    **[Decision, as a question]** — Chose: **[option]**. Why: [reason, or "user preference"]. Options were: [A / B]. _(Phase N · YYYY-MM-DD)_

**## Technical decisions (Claude's calls).** Non-obvious invisible-plumbing choices Claude made with no felt user difference (the silent half of the fork law). Format: Decision → Why → Alternatives considered. Example:

    **Used server-side pagination instead of cursor-based:** Simpler implementation for expected dataset size (<100k rows). Alternatives: cursor-based (better for real-time, overkill here), client-side (not viable at scale).

## Update Rules

1. **Read first.** Before touching code in an existing project, read the relevant living docs. Not the raw source files — the docs.
2. **Same session.** Update docs in the same session the code changed. Not in a follow-up session.
3. **Summaries only.** Docs are compressed — useful facts, not full code dumps or activity logs.
4. **No blanks.** An empty section means the doc wasn't written. Write it or delete the section heading.
5. **docs/api.md is always current.** If the API changed and docs/api.md wasn't updated, the doc is wrong — fix it before proceeding.
