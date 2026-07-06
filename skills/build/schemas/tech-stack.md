# Tech Stack — [Project Name]

## Choices
- **Language:**
- **Frontend framework:**
- **Backend framework:**
- **Database:**
- **Hosting / Deployment:**
- **Key libraries:**
- **Testing:**

## Constraints & Non-Negotiables
[Rules that apply from day one and cannot be broken. Examples: "strict TypeScript from commit 1", "no ORM — plain SQL only", "all dependencies pinned exactly, no ^ or ~ prefixes".]

- [Constraint]
- [Constraint]

## Explicit Exclusions
[What this stack deliberately does NOT use, and why. Be specific — exclusions prevent scope creep and wrong assumptions by agents.]

- **[Technology]:** [Why excluded]
- **[Technology]:** [Why excluded]

## Key Technical Decisions

| Decision | Why | Alternatives Rejected |
|----------|-----|-----------------------|
| [What was decided] | [Reason] | [What was considered and rejected] |

## Safety Defaults
[Machine-written by `/build-spec` (constitution mode) from the constitution-mode drilling session — the durable carrier for the 5 safety answers across phases. `/build-spec` reads it. Safe defaults if the drilling session omits a value: `production_domain: TBD`, `billing: no`, `sensitive_data: no`, `error_tracking: logs`, `deploy_platform: other`.]

```
production_domain: [value]
billing: [yes / planned / no]
sensitive_data: [yes / no]
error_tracking: [sentry / logs]
deploy_platform: [vercel / railway / render / other]
```

## Baselines
[Machine-written by `/build-spec` (constitution mode) from the confirmed baseline selection (mirrors the `baselines` array in `.build-state.json`); the fallback carrier when the state file is absent. List only confirmed keys.]

```
active: [app-shell, safety, repo, dx, observability]
```
