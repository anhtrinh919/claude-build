---
name: migrate
description: Brings a legacy build project onto the current /build stack — a v1 project (`foundationStatus` or `*-approved` steps) or a pre-rename build-2 project. Upgrades the state file, remaps the Design Tool, sweeps stale skill references, archives the old state. Trigger on /build:migrate.
user-invocable: true
---

# /build:migrate — bring a legacy project onto the current stack

The current stack reads `.build-state.json` on the current schema and ships as the `build` plugin. Two kinds of project need a bridge:

- **v1 `/build`** — a `.build-state.json` with an older shape (a `foundationStatus` field, `*-approved` steps, a three-track Design-Tool vocabulary) and docs naming retired sub-skills.
- **pre-rename `build-2`** — a `.build2-state.json`, already on the current schema, with docs naming `/build-2*`.

Both converge on the same target: `.build-state.json` on the current schema, plus docs that reference the current `/build*` skills and invoke `/build` without a filesystem path.

Runs **inline**. No subagents, no state of its own. It edits **only the project's own files** — never the `/build` skill definitions. It builds nothing, runs nothing, and commits nothing unless asked.

## Invocation contract

| Mode | Model | Mechanism | Reads | Writes | Terminal step |
|---|---|---|---|---|---|
| single | session model | inline | `.build-state.json` or `.build2-state.json`, `mission.md`, the project's own docs | the upgraded state file, a `.bak` of the original, remapped `mission.md ## Design Tool`, swept doc references | **none** — the user invokes `/build` when ready |

Run it once, in the project root. **Safest at a phase boundary** (`phase-complete` or `phase-blocked`), where nothing is half-built. Mid-phase works too — warn the user and let them choose, recommending proceed if the tree is committed and wait if `git status` is dirty.

## Mode detection

1. **`.build2-state.json` present** → **build-2 project**. Schema is current; only the filename and `/build-2*` references are stale.
2. Else **`.build-state.json` present**:
   - Has `foundationStatus`, **or** `step` ∈ {`frontend-complete`, `frontend-approved`, `constitution-approved`, `spec-approved`, `backend-approved`, `phase-approved`, bare `complete`} → **v1 project**.
   - Otherwise the schema is current. Grep the project's docs for a `build-` prefixed skill token (`/build-spec`, `/build-design`, …). Found → run **only** the reference sweep in Mode: v1 step 3; the state file needs nothing. None → **stop**: "Already on the current `/build` stack — just run `/build`."
3. Else **`mission.md` present** → between features. Write a fresh `.build-state.json` only if the user confirms they want `/build` to drive the next feature; else stop: "Nothing to migrate — just run `/build`."
4. Else → stop: "This isn't a build project."

## Mode: v1 project

### 1. Upgrade the state file, in place

| Field | Action |
|---|---|
| `phase`, `feature`, `reviewIteration`, `requirementsHash`, `currentSubStep`, `dogfoodPid` | carry over unchanged; default any missing (numbers → `0`, strings → `""`, `dogfoodPid` → `null`) |
| `step` | **translate**, below |
| `stack` | **write `"build"`** — the current schema requires it, v1 never wrote it |
| `foundationStatus`, `baselines` | **drop** — retired fields |

| Old v1 `step` | Current |
|---|---|
| `frontend-complete` / `frontend-approved` | `design-complete` |
| `constitution-approved` | `constitution-complete` |
| `spec-approved` | `spec-complete` |
| `backend-approved` | `backend-complete` |
| `phase-approved` | `phase-complete` |
| bare `complete` | `roadmap-complete` |
| any `*-complete` / `phase-blocked` / `roadmap-complete` | carry over unchanged |

`foundationStatus: "running"` means a background backend build may still be going. Tell the user to let it finish or discard that branch before migrating — the current stack has no background-foundation concept and will not adopt one.

### 2. Remap the Design Tool

`mission.md ## Design Tool` uses the old three-track vocabulary. Rewrite to the two current modes:

- starts with `claude-code` (any suffix — `-design`, `-taste`, `-impeccable`) → `claude-code`
- `external-*`, a tool name (`pencil`, `figma`, …), or anything else → `external`, appending `:<tool>` if the old value named one
- absent → leave absent; `/build:design` asks next phase.

### 3. Sweep stale skill references

A v1 project's docs — most critically its own `CLAUDE.md` — tell every fresh session to follow the *old* stack. Left in place they fight `/build` as conflicting instructions. Rewrite across `CLAUDE.md`, `product.md`, `backlog.md`, and every `specs/**/*.md` on disk:

| Old v1 token | Current |
|---|---|
| `/build-ba`, `/build-baseline` | `/build:spec` |
| `/build-frontend` | `/build:design` |
| `/build-sdd-review`, `/build-dogfood`, `/build-adversarial-review` | `/build:review` |
| `/build-write-tests` | `/build:backend` |
| `/build-bad-idea`, `/build-market-research` | drop the reference — both retired, and the current `/build:shape` does not judge ideas |
| `/build-spec`, `/build-design`, `/build-backend`, `/build-review`, `/build-shape`, `/build-polish`, `/build-migrate` | `/build:spec`, `/build:design`, `/build:backend`, `/build:review`, `/build:shape`, `/build:polish`, `/build:migrate` — the `build-` prefix was dropped; the plugin now qualifies each skill |
| any `~/.claude/skills/build/…` path | **invoke the `/build` skill** — it ships as a plugin, so no such path exists |

**Guards.** Apply each token with a trailing word boundary (`\b`) so a short token cannot corrupt a longer one — `/build-ba` must not eat `/build-backend`. Never touch a `/build` that is a filesystem path, a shell command, or a route (`dist/build`, `npm run build`). Stay idempotent: re-running changes nothing. Use a deterministic pass — one `perl -pi -e 's|OLD\b|NEW|g'` per mapping — and skip non-text files.

The project `CLAUDE.md` line

`` - `.build-state.json` holds the step. If it exists, `/build` orchestration is active — follow `~/.claude/skills/build/routing.md`. ``

becomes

`` - `.build-state.json` holds the step. If it exists, `/build` orchestration is active — invoke the `/build` skill. ``

### 4. Preserve

Copy the old `.build-state.json` to `.build-state.json.bak` before overwriting. Never delete it — it is the rollback.

## Mode: build-2 project

1. **Rename the state file.** Write `.build-state.json` from the contents of `.build2-state.json`, adding `stack: "build"` — the one field build-2 lacks. Rename the original to `.build2-state.json.bak`.
2. **Sweep `/build-2*` references** across `CLAUDE.md`, `product.md`, `backlog.md`, and every `specs/**/*.md`: `build-2` → `build` (which fixes `/build-2-spec`, `/build-2-review`, bare `/build-2`), `build2-state` → `build-state`, and any `~/.claude/skills/build-2/…` path → **invoke the `/build` skill**. Same guards as the v1 sweep.

## Report

Plain language (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`): that the project now runs on `/build`, which step it resumes from as a human phrase (`design-complete` → "the design is done; next is the backend build"), the Design-Tool value it mapped to, which docs were swept, and that the old state is backed up.

**Do not auto-run `/build`.** The user invokes it when ready, and it resumes from the mapped step.

### Worked example (v1)

`{ "phase": 2, "feature": "sharing", "step": "frontend-complete", "foundationStatus": "done", "baselines": ["observability"] }` becomes `{ "stack": "build", "phase": 2, "feature": "sharing", "step": "design-complete" }` with the other fields carried. `mission.md ## Design Tool: claude-code-design` → `claude-code`. Old file saved as `.build-state.json.bak`. Report: "Migrated — this project now runs on `/build`. It'll resume at the backend build for Phase 2 (sharing). Design set to Claude-designs-it. Your old state is backed up."

## Ground rules

1. Touches only the project's own files. Never edits a `/build` skill.
2. Builds nothing, runs nothing, commits nothing unless the user asks.
3. Idempotent — a project already on the current schema is a no-op with a plain "just run `/build`."
