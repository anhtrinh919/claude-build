# Living Docs — Structure & Update Rules

Living docs record project state, each at its own level of permanence. Agents read them before touching code, not raw source files.

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

Each file's purpose and authoring detail lives in its own schema under `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/` — this doc governs update cadence and cross-doc rules, not per-file content.

## Update cadence

**Always current — refreshed every phase wrap:** `product.md`, `tech-stack.md`, `roadmap.md`, `docs/api.md`, `docs/decisions.md`, `CLAUDE.md`, `README.md`, `CHANGELOG.md`.

**User-approved, frozen:** `outcome-card.md` — the only doc actually surfaced to the user verbatim and approved via `AskUserQuestion`. A later change restarts /build:spec phase mode.

**Agent-authored snapshot — Claude's draft, not user-approved; update without ceremony:** `mission.md` (a felt change still routes through the felt-impact-fork rule), `requirements.md` / `plan.md` / `validation.md` (each phase's own dated copy; `requirements.md` keeps its within-phase sha256 drift-hash), `design-brief.md`, `handover.md`, `design-comment.md`.

**Conditional:** `GLOSSARY.md` — write when vocabulary is genuinely ambiguous, same felt-need trigger as `design-brief.md`'s claude-code track. (`INTERACTIONS.md` — deliberate skip.)

## File Descriptions

### CLAUDE.md · product.md · backlog.md
Each has its own schema — read it there, not here: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/claude-md.md`, `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/constitution.md`, `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/backlog.md`.

### README.md
Project name, one-sentence mission, how to run locally, and a status line: "Phase N — [feature name] — in progress / complete." Update when a phase completes or setup changes.

### docs/architecture.md
Component map, data models with fields and types, tech decisions (link to docs/decisions.md for the why). Update when a component or the data model changes.

### docs/api.md
The live full API surface — method, path, auth, one line each. Update on every endpoint add, change, or removal. The spec file is only a phase's snapshot; this is the truth.

### docs/decisions.md
The project's **decision ledger** — the within-project analog of the brain. Re-read it before reopening any settled question, at phase start, and after compaction. Two sections:

**## User decisions (settled — do not reopen without asking).** Every fork the user resolved. One entry per decision — the question, the options offered, **what the user chose**, the one-line *why* (if stated), and the phase/date. An agent that believes a settled decision should now change must surface the prior choice to the user ("you chose X for this because Y — reopen?") — never quietly pick differently. Append-only; never delete a settled decision (mark it superseded if the user reopens it). Format:

    **[Decision, as a question]** — Chose: **[option]**. Why: [reason, or "user preference"]. Options were: [A / B]. _(Phase N · YYYY-MM-DD)_

**## Technical decisions (Claude's calls).** Non-obvious invisible-plumbing choices Claude made with no felt user difference (the silent half of the fork law). Format: Decision → Why → Alternatives considered.

## Update Rules

1. **Read first.** Before touching code in an existing project, read the relevant living docs. Not the raw source files — the docs.
2. **Same session.** Update docs in the same session the code changed. Not in a follow-up session.
3. **Summaries only.** Docs are compressed — useful facts, not full code dumps or activity logs.
4. **No blanks.** An empty section means the doc wasn't written. Write it or delete the section heading.
5. **docs/api.md is always current.** If the API changed and docs/api.md wasn't updated, the doc is wrong — fix it before proceeding.
