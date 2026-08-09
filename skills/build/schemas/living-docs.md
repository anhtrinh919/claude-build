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
    │       ├── design-notes.md
    │       ├── design-tokens.css
    │       ├── mockups/
    │       └── handover.md
    └── docs/
        ├── architecture.md
        ├── api.md
        ├── rejected.md

Each file's purpose and authoring detail lives in its own schema under `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/` — this doc governs update cadence and cross-doc rules, not per-file content.

## Update cadence

**Always current — refreshed every phase wrap:** `product.md`, `tech-stack.md`, `roadmap.md`, `docs/api.md`, `CLAUDE.md`, `README.md`, `CHANGELOG.md`.

**Append-only, never refreshed:** `docs/rejected.md` — it records history, not state, so nothing in it can go out of date.

**User-approved, frozen:** `outcome-card.md` — the only doc actually surfaced to the user verbatim and approved via `AskUserQuestion`. A later change restarts /build:spec phase mode.

**Agent-authored snapshot — Claude's draft, not user-approved; update without ceremony:** `mission.md` (a felt change still routes through the felt-impact-fork rule), `requirements.md` / `plan.md` / `validation.md` (each phase's own dated copy; `requirements.md` keeps its within-phase sha256 drift-hash), `design-brief.md`, `design-notes.md`, `handover.md`, `design-comment.md`.

**Conditional:** `GLOSSARY.md` — write when vocabulary is genuinely ambiguous, same felt-need trigger as `design-brief.md`'s claude-code track. (`INTERACTIONS.md` — deliberate skip.)

## File Descriptions

### CLAUDE.md · product.md · backlog.md
Each has its own schema — read it there, not here: `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/claude-md.md`, `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/constitution.md`, `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/backlog.md`.

### README.md
Project name, one-sentence mission, how to run locally, and a status line: "Phase N — [feature name] — in progress / complete." Update when a phase completes or setup changes.

### docs/architecture.md
Component map, data models with fields and types, and the component-level and data-model calls with their why and the alternatives rejected. Update when a component or the data model changes.

### docs/api.md
The live full API surface — method, path, auth, one line each. Update on every endpoint add, change, or removal. The spec file is only a phase's snapshot; this is the truth.

### docs/rejected.md
The project's **rejection ledger** — the one thing no other living doc records: which options a resolved fork killed, so nobody proposes them twice. Read it before surfacing any fork. Never read it for what the product does.

**An entry describes a question. A living doc describes the product. No sentence does both.** That is the whole scope, and it makes a conflict with a living doc impossible rather than merely unlikely — the two never claim the same fact. One section, one format, one line per entry:

    **[Question]** — Rejected: [A], [B]. _(Phase N · YYYY-MM-DD)_
    **[Question]** — Rejected: [A]. _(Phase 1 · YYYY-MM-DD · reopened Phase 3)_

Four rules, all load-bearing:

1. **One line per entry. No exceptions.** A rejection needing a paragraph is not a rejection — it is product state, and it belongs to the doc that owns it.
2. **Nothing in the file but entries.** No prose sections, no phase headings, no narration.
3. **Never the chosen option, never a mechanism, never a why that will expire.** The choice lives where it is implemented. A reason that generalizes is a standing preference → `CLAUDE.md`.
4. **Append-only.** A reopened question gains `· reopened Phase N` on its existing line. Never rewrite an entry, never mark one superseded — a rejected option stays rejected whatever the product later does.

**What does NOT go here**, each with its real home: phase narration, implementation notes, review findings and fixes → `CHANGELOG.md` · stack, dependency, and library calls → `tech-stack.md ## Key Technical Decisions` (Decision / Why / Alternatives Rejected) · component and data-model calls → `docs/architecture.md` · a standing user directive → `CLAUDE.md` · a phase's structural design notes → `specs/<phase>/design-notes.md`.

## Update Rules

1. **Read first.** Before touching code in an existing project, read the relevant living docs. Not the raw source files — the docs.
2. **Same session.** Update docs in the same session the code changed. Not in a follow-up session.
3. **Summaries only.** Docs are compressed — useful facts, not full code dumps or activity logs.
4. **No blanks.** An empty section means the doc wasn't written. Write it or delete the section heading.
5. **docs/api.md is always current.** If the API changed and docs/api.md wasn't updated, the doc is wrong — fix it before proceeding.
