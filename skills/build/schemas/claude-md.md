# [Project Name] — Agent Operating Brief

> This is the project's `CLAUDE.md` — auto-loaded by Claude Code at the start of every session in this directory. It is the **index + operating context** for the project. It does NOT duplicate the living docs — it points to them. On any conflict, the doc it points to wins.

[One sentence: what this product is and who it's for — drawn from `mission.md`.]

This project is built by the `/build` SDD stack. **Before touching code, read the living doc that covers your task — never scan raw source when a doc already covers it.**

## Resuming a build
- `.build-state.json` holds the current `step` + `phase`. If it exists, `/build` orchestration is active — follow `${CLAUDE_PLUGIN_ROOT}/skills/build/SKILL.md`.
- User gates are outcome-only: the user approves the product story, each phase's Outcome Card, the design, and the phase-end dogfood. The user never approves spec files.

## Living docs — read the one that fits
- `mission.md` / `product.md` — what the product is, for whom; screen inventory, named flows, App Map.
- `tech-stack.md` — stack + non-negotiables (e.g. strict TypeScript, pinned deps).
- `roadmap.md` — ordered phase list.
- `specs/YYYY-MM-DD-<feature>/` — per-phase `requirements.md` · `plan.md` · `validation.md` (frozen after spec approval).
- `docs/decisions.md` — technical forks + why + alternatives rejected.
- `docs/architecture.md` / `docs/api.md` — current component map & live API surface.
- `backlog.md` — short-term task lake: roll-in candidates, dogfood polish, side tasks. Transient — things to do that don't fit the current phase's spec. Read at phase start; appended any time the user defers a request mid-build.

## Project directives
> The home for durable, project-specific instructions, preferences, and constraints the user gives **in conversation** — the things with no place in the structured docs above. **Agents: append a dated one-liner here the moment the user states a durable project-scoped directive.** This is NOT for technical decisions (→ `docs/decisions.md`) — it's for how the user wants *this project* run.
>
> Examples: "always format money as VND, no decimals" · "never auto-deploy — the user deploys manually" · "reuse the design system in `/shared`, don't invent components" · "primary users are store managers aged 40+, keep copy plain" · "the stakeholder is non-technical — no jargon in any UI or email."

- _(none yet — append as they surface)_
