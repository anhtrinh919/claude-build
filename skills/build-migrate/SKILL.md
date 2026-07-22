---
name: build-migrate
description: >
  Bring a legacy build project onto the current `/build` stack. Handles two origins: an **old v1 /build** project — a `.build-state.json` carrying the v1 shape (`foundationStatus` / `frontend-complete` / `*-approved`) and references to retired sub-skills — upgraded in place; or a **pre-rename build-2** project — a `.build2-state.json` — whose schema is already current and only needs the filename and stale `/build-2*` references fixed. One command: detects which, brings the state file to the current schema, remaps `mission.md`'s Design Tool to the two current tracks, sweeps stale skill references across the project's own docs (`CLAUDE.md`, `product.md`, `backlog.md`, `specs/**/*.md`) to the current `/build*` skills, archives the old state as a `.bak`, and reports what moved in plain language. Touches ONLY the project's own files — never the `/build` skills. Safest at a phase boundary (`phase-complete`). Trigger on /build-migrate, "migrate this project to build", "move this onto the build stack", or whenever a project carries a `.build2-state.json` or a v1-shaped `.build-state.json`.
---

# /build-migrate — bring a legacy build project onto the current /build stack

The current `/build` stack reads `.build-state.json` (current schema) and its skills ship as the `build` plugin. Two kinds of legacy projects need a bridge:

- **v1 `/build`** — wrote `.build-state.json` with a slightly older shape (a renamed step, a dropped `foundationStatus`, a three-track Design-Tool vocabulary) and its docs name retired sub-skills (`/build-ba`, `/build-frontend`, `/build-dogfood`, `/build-sdd-review`, …) and a `~/.claude/skills/build/routing.md` path.
- **pre-rename `build-2`** — wrote `.build2-state.json` (already the current schema) and its docs name `/build-2*` skills and a `~/.claude/skills/build-2/SKILL.md` path.

This skill converges either onto the current world: `.build-state.json` (current schema) + docs that reference the current `/build*` skills and invoke `/build` without a filesystem path. It edits **only the project's own files** — never the `/build` skill definitions.

Runs **inline**. No subagents, no state of its own.

## When it applies & phase-boundary safety
- Run it once, in the project root.
- **Safest at a phase boundary** (`step: phase-complete` / `phase-blocked`): nothing is half-built. Mid-phase works too, but warn the user and let them choose (`AskUserQuestion`, recommend proceed if the tree is committed, wait if `git status` is dirty).

## Detect the case (ladder)
1. **`.build2-state.json` present** → **Case B** (renamed build-2 project). Schema already current — only the filename and `/build-2*` doc references are stale.
2. Else **`.build-state.json` present**:
   - Has a `foundationStatus` field, **or** `step` ∈ {`frontend-complete`, `frontend-approved`, `constitution-approved`, `spec-approved`, `backend-approved`, `phase-approved`, bare `complete`} → **Case A** (v1 `/build` project) — needs a schema upgrade.
   - Otherwise it is already the current schema → **stop**: "Already on the current `/build` stack — just run `/build`." (no-op keeps the skill idempotent).
3. Else **`mission.md` present** → between features; write a fresh `.build-state.json` only if the user confirms they want `/build` to drive the next feature, else stop: "Nothing to migrate — just run `/build`."
4. Else → stop: "This isn't a build project."

---

## Case A — v1 /build → current

### 1. Upgrade the state file (in place; filename stays `.build-state.json`)
Read `.build-state.json`; rewrite it as:

| Field | Action |
|---|---|
| `phase`, `feature`, `reviewIteration`, `requirementsHash`, `currentSubStep`, `dogfoodPid` | carry over unchanged (default any missing: numbers→`0`, strings→`""`, `dogfoodPid`→`null`) |
| `step` | **translate** (table below) |
| `stack` | **write** `"build"` — current schema requires it, v1 never wrote it |
| `foundationStatus`, `baselines` | **drop** — retired fields, not part of the current schema |

**Step translation:**
| Old v1 `step` | Current `step` |
|---|---|
| `frontend-complete` / `frontend-approved` | `design-complete` |
| `constitution-approved` | `constitution-complete` |
| `spec-approved` | `spec-complete` |
| `backend-approved` | `backend-complete` |
| `phase-approved` | `phase-complete` |
| bare `complete` | `roadmap-complete` |
| any `*-complete` / `phase-blocked` / `roadmap-complete` | carry over unchanged |

If old `foundationStatus` is `"running"`: a background backend build may still be going — tell the user to let it finish (or discard that branch) before migrating; the current stack has no background-foundation concept and won't adopt it.

### 2. Remap the Design Tool
In `mission.md`, the `## Design Tool` value uses the old three-track vocabulary. Rewrite to the two current tracks:
- starts with `claude-code` (any suffix — `-design` / `-taste` / `-impeccable`) → `claude-code`
- `external-*`, a tool name (`pencil` / `figma` / …), or anything else → `external` (append `:<tool>` if the old value named one)
- absent → leave absent (`/build-design` will ask next phase).

### 3. Sweep stale skill references in the project's own docs
A v1 project's docs — most critically its own `CLAUDE.md` — tell every fresh session to follow the *old* stack; left in place they fight `/build` as conflicting instructions. Rewrite across `CLAUDE.md`, `product.md`, `backlog.md`, and every `specs/**/*.md` on disk (all phases):

| Old v1 token | Current |
|---|---|
| `/build-ba`, `/build-baseline` | `/build-spec` |
| `/build-frontend` | `/build-design` |
| `/build-sdd-review`, `/build-dogfood`, `/build-adversarial-review` | `/build-review` |
| `/build-bad-idea` | `/build-shape` |
| `/build-write-tests` | `/build-backend` |
| `/build-market-research` | drop the reference (skill retired) |
| any `~/.claude/skills/build/…` path (e.g. `routing.md`) | **invoke the `/build` skill** (path-independent — it ships as a plugin, no `~/.claude/skills/build/`) |

`/build-spec`, `/build-design`, `/build-backend`, `/build-polish` keep their names (same in both stacks) — leave them. `.build-state.json` is unchanged in Case A.

**Guards:** apply the specific tokens with a trailing word-boundary (`\b`) so a short token can't corrupt a longer one (`/build-ba` must not eat `/build-backend`); never touch a `/build` that is a filesystem path or shell command (`dist/build`, `npm run build`, a `/build` route); idempotent (re-running changes nothing). Do it with a deterministic pass (one `perl -pi -e 's|OLD\b|NEW|g'` per mapping) or careful edits; skip non-text/binary files.

The project `CLAUDE.md` "Resuming a build" line
`` - `.build-state.json` holds the step. If it exists, `/build` orchestration is active — follow `~/.claude/skills/build/routing.md`. ``
must become
`` - `.build-state.json` holds the step. If it exists, `/build` orchestration is active — invoke the `/build` skill. ``

### 4. Preserve
Copy the old `.build-state.json` to `.build-state.json.bak` before overwriting (never delete it — it's the rollback).

---

## Case B — renamed build-2 → current

The schema is already correct apart from `stack` (build-2 predates it); the filename and `/build-2*` references are stale.

### 1. Rename the state file
Write `.build-state.json` from the contents of `.build2-state.json` (adding `stack: "build"`, the one field build-2 lacks), then rename the original to `.build2-state.json.bak` (the rollback).

### 2. Sweep `/build-2*` references in the project's own docs
Across `CLAUDE.md`, `product.md`, `backlog.md`, and every `specs/**/*.md`, apply the same rename that renamed the stack itself:
- `build-2` → `build` (fixes `/build-2-spec`→`/build-spec`, `/build-2-review`→`/build-review`, bare `/build-2`→`/build`, etc.)
- `build2-state` → `build-state`
- any `~/.claude/skills/build-2/…` path → **invoke the `/build` skill** (path-independent).

Same guards as Case A. `.build-state.json` needs no `.bak` here beyond the renamed original.

---

## Report
Plain language (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md` when present, else plain): what the project is now on `/build`, which step it will resume from (translate the step to a human phrase — e.g. `design-complete` → "the design is done; next is the backend build"), the Design-Tool value it mapped to (Case A), which docs were swept clean, and that the old state is saved as a backup. Do **not** auto-run `/build` — the user invokes it when ready; it resumes from the mapped step.

## Worked example (Case A)
Old `.build-state.json`:
```
{ "phase": 2, "feature": "sharing", "step": "frontend-complete",
  "reviewIteration": 0, "requirementsHash": "a1b2…", "currentSubStep": null,
  "dogfoodPid": null, "foundationStatus": "done", "baselines": ["observability"] }
```
→ new `.build-state.json`:
```
{ "stack": "build", "phase": 2, "feature": "sharing", "step": "design-complete",
  "reviewIteration": 0, "requirementsHash": "a1b2…", "currentSubStep": null,
  "dogfoodPid": null }
```
(`frontend-complete`→`design-complete`; `foundationStatus`/`baselines` dropped; `stack: "build"` written; rest carried.) `mission.md ## Design Tool: claude-code-design` → `claude-code`. Old file saved as `.build-state.json.bak`. Report: "Migrated — this project now runs on `/build`. It'll resume at the backend build for Phase 2 (sharing). Design track set to Claude-designs-it. Your old state is backed up."

## Boundaries
- Touches only the project's own files (`.build-state.json` / `.build2-state.json`, the `.bak`, `mission.md ## Design Tool`, and the reference sweep across `CLAUDE.md` / `product.md` / `backlog.md` / `specs/**/*.md`). Never edits any `/build` skill.
- Doesn't build anything, doesn't run `/build`, doesn't commit unless the user asks.
- Idempotent: already on the current schema → no-op with a plain "just run `/build`."
