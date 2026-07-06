# Observability Baseline

Standard monitoring and logging every production app must include. Answers the question: "how do you know when it breaks, and how do you find out why?" Referenced by `/build-spec` when `observability` is in the active baselines list.

**When this applies:**
- First phase of a project with a backend → SCAFFOLD all items
- Subsequent phases → INHERIT; no re-scaffold
- Static sites / no-backend projects → skip (nothing to observe server-side)

**Stack-agnostic rule:** Requirements below describe WHAT is needed. Claude resolves the specific library from `tech-stack.md`: Node → pino/winston + Sentry Node SDK; Python → structlog + Sentry Python SDK; Go → zap + Sentry Go SDK; etc.

**Reads from `tech-stack.md ## Safety Defaults`:** The `error_tracking` answer from `production-safety-baseline.md`'s Tier 2 (SD4) determines which error tracking path is scaffolded here. No new question needed.

---

## Tier 1 — Auto-Scaffold

### Structured logging

Replace all bare `console.log` / `print` / `fmt.Println` with a structured logger that outputs JSON in production and human-readable format in development. Environment-controlled via `LOG_FORMAT=json|pretty` env var.

**What to log (always):**
- Request: method, path, status code, duration (ms), request ID
- Errors: message, stack trace, request ID, user ID (never PII)
- Application events: startup, shutdown, DB connection established/lost
- Business events relevant to debugging: "payment webhook received", "user created", "job queued"

**What NEVER to log:**
- Passwords, tokens, secrets, API keys (even partially — truncation is not safe)
- Full request bodies (may contain passwords, credit card numbers, PII)
- PII: email addresses, phone numbers, names, addresses — log user ID only
- Session cookies or auth headers

**Log levels — use them correctly:**
- `debug` — detailed internal state, dev only (never in production)
- `info` — normal operations, request lifecycle, startup
- `warn` — degraded but not broken (3rd-party timeout, retry triggered, non-critical missing config)
- `error` — something broke and needs attention; always include stack trace

**Request ID propagation:**
- Generate a UUID per incoming request
- Attach to all log lines for that request (logger context / async-local-storage)
- Return in response header: `X-Request-ID`
- If request carries an inbound `X-Request-ID` (from upstream), use that instead

### Error tracking (path depends on SD4 answer)

**If Sentry:**
- SDK initialized at server root, before routes
- `Sentry.captureException(err)` in the global error handler (catch-all)
- `SENTRY_DSN` in `.env.example`
- Source maps uploaded in CI build step so Sentry shows original TypeScript/source line numbers, not minified bundle
- Set `environment` tag from `NODE_ENV` so errors are tagged production/staging/development
- Alert rule: email on first occurrence of new error, daily digest on recurring

**If logs-only:**
- Structured JSON logger configured as above
- Note in README: "Errors log to stdout — check platform log viewer"
- Warn in setup docs: log retention varies by platform (Vercel: 1 hour on free tier; Railway: 7 days; Render: depends on plan)

### Health check (upgraded from production-safety-baseline)

`GET /api/health` returns a deep readiness check — not just "server is up" but "all dependencies are reachable."

```json
// HTTP 200 — healthy
{
  "status": "ok",
  "ts": "2026-06-17T10:30:00.000Z",
  "checks": {
    "db": "ok",
    "latency_ms": 4
  }
}

// HTTP 503 — degraded (triggers platform rollback)
{
  "status": "degraded",
  "ts": "2026-06-17T10:30:00.000Z",
  "checks": {
    "db": "error",
    "latency_ms": null
  }
}
```

Implementation: run `SELECT 1` (or DB equivalent) with a 2-second timeout. If it fails or times out → `db: "error"` → HTTP 503. No auth required on this endpoint.

When both `safety` and `observability` are active, `/build-spec` removes the shallow health check line from within the S6 block and injects O4 instead. The S6 pagination check remains. `/build-spec` is the sole executor of this substitution.

### Uptime monitoring

After deploy, configure UptimeRobot (free tier) or equivalent to monitor `/api/health` every 5 minutes. Alert to the owner's email on downtime.

Add setup instructions to README under a `## Monitoring` section:
```
1. Sign up at uptimerobot.com (free)
2. Add HTTP(S) monitor → URL: https://[your-domain]/api/health
3. Check interval: 5 minutes
4. Alert: email notification on down
```

This is a deploy-time setup step, not automatable at scaffold time. Flagged in the deploy-time checklist (O4 below).

### Error boundary integration with Sentry (ui: true + Sentry only)

If the project has a UI and uses Sentry: wire React (or framework equivalent) error boundaries to `Sentry.captureException`. This catches client-side crashes that the server never sees.

The app-shell spec (`build-design/references/app-shell-spec.md`) requires route-level and component-level error boundaries. These same boundaries should call `Sentry.captureException(error)` in their `componentDidCatch` / `onError` handlers — not just render a fallback UI.

---

## Tier 2 — Ask Once

No new questions. Reads `error_tracking` from `tech-stack.md ## Safety Defaults` (collected by `production-safety-baseline.md` SD4).

---

## Tier 3 — Observability Validation Block (injected into `validation.md`)

```markdown
## Observability Baseline Checks

### O1 — Structured logging [static]
- [ ] No bare `console.log` in `src/`: `grep -rn "console\.log" src/ --include="*.ts" --include="*.js" --exclude-dir=scripts --exclude-dir=seeds` returns empty. If any matches exist, they are blocking failures — there is no "intentional debug log" exception. Sanctioned logging must use the structured logger.
- [ ] Structured logger configured with log levels (`debug`, `info`, `warn`, `error`)
- [ ] `LOG_FORMAT` env var documented in `.env.example`

### O2 — Request logging [runtime]
- [ ] Every HTTP request produces a log line with: method, path, status, duration
- [ ] Log line includes a request ID
- [ ] Response includes `X-Request-ID` header: `curl -I <health-endpoint> | grep -i x-request-id` returns a UUID

### O3 — Error tracking [runtime / deploy-time]
If Sentry:
- [ ] SDK initialized: `grep -r "Sentry.init" src/` returns a result
- [ ] Global error handler calls `Sentry.captureException`: `grep -r "captureException" src/` returns a result
- [ ] `SENTRY_DSN` in `.env.example`
- [ ] [deploy-time] Test event received in Sentry dashboard after triggering a deliberate 500 error

If logs-only:
- [ ] `error` log level used in global error handler (not `console.error`)
- [ ] README documents where to find logs on the chosen platform

### O4 — Health check (deep) [runtime]
- [ ] `GET /api/health` returns `{status, ts, checks: {db}}` — not just `{status: "ok"}`
- [ ] Health returns HTTP 503 when DB is unreachable: kill the DB connection temporarily, confirm 503 (or test with wrong DATABASE_URL in a test env)
- [ ] [deploy-time] UptimeRobot (or equivalent) configured and receiving pings from production health endpoint
```

---

## What this does NOT cover

- Distributed tracing (OpenTelemetry, Jaeger) — post-PMF, needed when multiple services exist
- APM / performance monitoring (New Relic, Datadog) — post-PMF
- Log aggregation services (Papertrail, Logtail) — optional; platform logs cover the MVP floor
- Alerting beyond error tracking (PagerDuty, OpsGenie) — post-PMF
- Custom business metrics dashboards — post-PMF
