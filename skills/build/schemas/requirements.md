# [Feature Name] Requirements

---
phase: [N]
type: initial | feature | rebuild
ui: true | false
---

## Phase type

- **`initial`** — greenfield; no existing UI patterns to honor.
- **`feature`** — adds capability to an existing product; follow existing patterns.
- **`rebuild`** — visual/structural redesign; existing UI patterns explicitly overridden by the design file.

## UI flag

- **`ui: true`** (default) — phase produces visible UI. `/build-review` (pipeline-review mode) Round 2 runs.
- **`ui: false`** — pure backend/infra/tooling. Round 2 skipped; Round 1 still runs. Do not infer from screen count — set the flag.

## Scope
[What this phase delivers. What a user can do on completion that they couldn't before. One paragraph.]

## User Stories
- As a [actor], I can [specific action] so that [specific outcome].

[One story per major user action. "Manage content" is not a story — name the exact action.]

## UI Requirements
Every screen in this phase. Every unique state = its own row.

| Screen | State | Key UI Elements | Primary User Action |
|--------|-------|-----------------|---------------------|
| [Name] | Default | [Main elements] | [What user does] |
| [Name] | Empty | [Empty state + action prompt] | |
| [Name] | Loading | [Skeleton or spinner] | |
| [Name] | Error | [Error message + recovery] | |
| [Name] | Mobile | [Intentional layout adaptation] | |

## Data Model
[Tables or schemas with field names and types. Include relationships.]

```
[Table / Schema name]
- field_name: type — description
- field_name: type — description
```

## API Contracts
One section per endpoint. Frontend and backend both build against these exactly. Every endpoint names its consuming screen(s); every screen with data needs has a backing endpoint. `/build-spec` reconciles both ways — an endpoint no screen consumes (and isn't `internal`), or a screen with no backing endpoint, is a spec error caught before the build.

### [Endpoint Name]
- **Method + path:** `[GET/POST/PUT/DELETE] /api/[path]`
- **Consumed by:** [screen name(s) from the UI Requirements table that call this — or `internal` for endpoints with no UI consumer, e.g. webhooks, cron, server-to-server]
- **Auth required:** Yes / No
- **Request body:** `{ field: type }` (POST/PUT only)
- **Query params:** `?field=type` (GET only)
- **Success response:** `{ field: type }` — status [200/201]
- **Error responses:**
  - `400`: [specific condition] — `{ error: "message" }`
  - `401`: Unauthenticated
  - `404`: [resource not found condition]
  - `500`: Unexpected server error

## Constraints & Context
[Business rules, tone, patterns to follow from tech-stack.md, non-negotiables for this phase.]

- [Constraint]
- [Pattern to follow from existing codebase]

## App Shell

> **Phase 0 (`initial`):** Shell is built in this phase (part of the Foundation). Fill each subsection from `/build-design`'s app-shell spec reference (`${CLAUDE_PLUGIN_ROOT}/skills/build-design/references/app-shell-spec.md`), adapted to this app's specific context (section labels, icon choices, social login providers, which settings categories apply).
> **Phase 1+ (`feature`):** Shell is inherited from Phase 0 — do not rebuild. Note only what this phase adds or changes (e.g. a new nav item, a new settings category). Mark unchanged items "inherited".
> **Any phase (`rebuild`):** Shell is explicitly redesigned — fill from scratch, overriding Phase 0's shell.
> **`ui: false`:** Delete this section entirely.

### Navigation
[Describe which responsive nav pattern applies (sidebar / bottom nav / rail) per the baseline. Specify section labels, item order, and icon choices for this app. Note any deviation from baseline with a reason.]

### Auth
[Confirm auth gate applies: yes/no. Name the social login providers to include (e.g. Google only, or Google + GitHub). Note any custom redirect logic or session expiry behavior.]

### Settings
[List which settings categories are in scope. Categories from the baseline that are not relevant to this app's feature set may be omitted — name them and explain why (e.g. "Billing — omitted, this app has no paid plans").]

### Universal Patterns
[Confirm toast system, skeleton loading, error boundaries, and empty states are all in scope. Note any deviation from baseline — e.g. "No notifications bell — this app has no async background activity".]

## Excluded from This Phase
Explicitly named. Anything not listed above is out of scope.

- [Feature or behavior explicitly excluded]
- [Another explicit exclusion]
