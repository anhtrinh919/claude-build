# claude-build

**Spec-Driven Development for non-devs.** A [Claude Code](https://claude.com/claude-code) plugin that turns an idea into a shipped, dogfooded product one vertical slice at a time — gating the build so the AI can't steamroll you with unreviewed decisions.

You describe what you want. `/build` runs a disciplined pipeline: it pressure-tests the idea, drills you for a product constitution, then builds each phase as a real end-to-end slice — spec → design → backend → review — pausing only at outcome-level go/no-go gates. It resumes mid-phase after a context reset, because the state lives in a file, not the chat.

## What you get

One command, `/build`, orchestrating eight skills:

| Skill | Role |
|---|---|
| `build` | The orchestrator — state-file-driven, re-entrant pipeline. |
| `build-shape` | Front gate: validates the idea is worth building, surfaces the big forks, pressure-tests the product shape. |
| `build-spec` | Drills you and authors the specs — constitution + per-phase requirements / plan / validation. |
| `build-design` | Designs the UI (in-code mockups) or reviews your external design, against one UX / craft / accessibility gate. |
| `build-backend` | Wave-dispatched implementation; every API contract integration-tested. |
| `build-review` | Code review + a real browser dogfood, with silent auto-fix. |
| `build-polish` | Batch backlog drainer for bugs and small improvements. |
| `build-migrate` | Upgrades a project built on an older version of the stack. |

## Install

```
/plugin marketplace add anhtrinh919/claude-build
/plugin install build@claude-build
```

Then start a build:

```
/build            # resume where you left off
/build <idea>     # kick off a new project from a one-line brief
```

## Requirements

- **Claude Code** — the plugin host.
- **A browser-automation tool exposed as the `/browse` skill.** The `/build-review` phase dogfoods your app in a real browser; without a `/browse` tool installed, the review phase's interactive walk can't run (the code review, type-checks, and API-contract checks still do).
- **Optional — `bun` + a local "brain" memory system** at `$HOME/.claude/brain`. If present, the skills pull relevant past learnings to sharpen their judgment; if absent, they skip that step cleanly. Not required.

## How it gates you

Every *felt* product decision is surfaced as a fork you pick — never silently assumed. Every phase ends at an outcome-level gate ("does this do what you wanted?"), not a technical one. Machine-checkable artifacts (specs, types, tests) are validated automatically, so you're only asked about things you can actually judge.

## License

MIT © Anh Trinh
