# [Feature Name] Requirements

---
phase: [N]
ui: true | false
---

## UI flag

`ui: true` (default) runs `/build:review`'s dogfood step; `ui: false` skips it. Set the flag — do not infer it from screen count.

## Scope
[What this phase delivers. One paragraph.]

## User Stories
- As a [actor], I can [specific action] so that [specific outcome].

[One story per user action — name the exact action.]

## UI Requirements
Every screen; each state is its own row.

| Screen | State | Key UI Elements | Primary User Action |
|--------|-------|-----------------|---------------------|
| [Name] | Default | [Main elements] | [What user does] |
| [Name] | Empty | [Empty state + action prompt] | |
| [Name] | Loading | [Skeleton or spinner] | |
| [Name] | Error | [Error message + recovery] | |
| [Name] | Mobile | [Intentional layout adaptation] | |

## Data Model
[Tables or schemas: field names, types, relationships.]

```
[Table / Schema name]
- field_name: type — description
- field_name: type — description
```

## API Contracts
One section per endpoint. Frontend and backend build against these exactly. Every endpoint names its consuming screen(s); every data screen has a backing endpoint. `/build:spec` reconciles both: an unused endpoint (not `internal`), or an unbacked screen, is a spec error.

### [Endpoint Name]
- **Method + path:** `[GET/POST/PUT/DELETE] /api/[path]`
- **Consumed by:** [screen name(s) that call this, or `internal` for no UI consumer, e.g. webhooks, cron]
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
[Business rules, tone, patterns from tech-stack.md.]

- [Constraint]
- [Pattern to follow from existing codebase]

## App Shell

> **Phase 0:** Shell is built this phase. State each decision below for this app. There is no standard shell to copy — pick what this product needs.
> **Phase 1+:** Shell is inherited from Phase 0 — do not rebuild it. Note only what changes; mark unchanged items "inherited". A redesign fills it from scratch.
> **`ui: false`:** Delete this section entirely.

### Navigation
[Name the nav pattern and why it fits this app. Set section labels, item order, and behavior at each breakpoint you support.]

### Auth
[Confirm the auth gate: yes/no. Name the login methods. Note redirects and session expiry.]

### Settings
[List the settings categories in scope. Omit categories that don't apply — name them and say why (e.g. "Billing — omitted").]

### Universal Patterns
[Name the feedback, loading, error, and empty-state patterns this app uses, and where each appears. Only what this phase actually needs.]

## Excluded from This Phase
Explicitly named. Anything not listed above is out of scope.

- [Feature or behavior explicitly excluded]
- [Another explicit exclusion]
