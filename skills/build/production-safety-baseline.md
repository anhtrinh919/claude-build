# Production Safety Baseline

Engineering hygiene every project must include. Sister doc to the app-shell spec (UI/UX — now `/build-design`'s app-shell spec reference, `${CLAUDE_PLUGIN_ROOT}/skills/build-design/references/app-shell-spec.md`), `repo-baseline.md` (project management), `dx-baseline.md` (tooling), and `observability-baseline.md` (monitoring). Referenced by `/build-spec` when `safety` is in the active baselines list.

**When this applies:**
- First phase of a project (detected by absence of `## Safety Defaults` in `tech-stack.md`) → SCAFFOLD all Tier 1 items + inject full safety validation block
- Subsequent phases → INHERIT; add only new endpoints from this phase to S3/S4 checks
- `rebuild`-type phase → re-verify all Tier 1 items still in place + new endpoints

**Applicability gate — skip server/DB/API sections for:**
- Static sites or SPAs with no backend (all routes served by CDN, no server process)
- `ui: false` tools with no HTTP server
- These still apply: secret hygiene (S1), gitignore scaffold

**Stack-agnostic:** requirements describe WHAT, not HOW. Resolve the specific library from `tech-stack.md` — e.g. "security headers middleware" → `helmet` (Express), `secure_headers` (Flask), Next.js `headers()` config. Never assume Node.

---

## Tier 1 — Auto-Scaffold

Generate these for every qualifying project. No questions asked. No user approval. Not listed in latent decisions.

### Secret hygiene
- `.gitignore` — must include: `.env*`, `node_modules/`, `dist/`, `build/`, `.next/`, `.DS_Store`, `*.log`, `.vercel/`, stack-specific build artifacts
- `.env.example` — all config keys, no values; a comment per key describing what it is and where to get it
- **CI/server-side secret scanning** — add a GitHub Actions step (or platform equivalent) that scans commits for secret patterns before merge. Client-side pre-commit hooks help but are bypassable with `--no-verify`; server-side is the enforceable layer. Patterns: `sk-`, `ghp_`, `AKIA`, `postgres://`, `mysql://`, `mongodb+srv://`, generic high-entropy token patterns

### Auth & session hygiene (applies if project has any user authentication)
- **Password hashing** — bcrypt (cost ≥ 12) or argon2id. Never plaintext, MD5, SHA1, or unsalted SHA256.
- **Session/JWT secret** — cryptographically secure random (min 256 bits), in env var (`SESSION_SECRET` / `JWT_SECRET`), never hardcoded.
- **Secure cookie flags** — `httpOnly: true`, `secure: true` (production), `sameSite: 'Strict'` (or `'Lax'` for OAuth).
- **Authorization model** — before any resource endpoint, document ownership in the spec: `owner-only`, `team-scoped`, or `role-based`. BOLA pattern `resource.userId !== session.userId` is only correct for `owner-only`. Document in `docs/decisions.md`.

### Backend security (applies if project has an HTTP server)
- **Security headers middleware** — single middleware at server root, before any routes. On `ui: true` projects, configure cross-origin asset loading explicitly — Helmet's default `Cross-Origin-Resource-Policy: same-origin` blocks CDN-served images and fonts.
- **CORS** — restrictive by default. Dev: `localhost` at dev port. Prod: domain from `tech-stack.md ## Safety Defaults → production_domain`. If TBD: use `CORS_ORIGIN` env var (S2 check blocks until set). Never `*` on authenticated endpoints. Allowlist from environment, not hardcoded.
- **Rate limiting on auth endpoints** — login, register, password reset, token-issuing routes. HTTP 429. Initial: 10 req / 15 min / IP. Configure `trust proxy` correctly behind Vercel/Cloudflare/nginx.
- **Request body size limit** — at server root: 1MB JSON, 10MB file uploads. Without this, large payloads exhaust server memory (trivial DoS).
- **Server-side input validation** — every mutation endpoint validates all body fields for type, length, format before business logic. Invalid → HTTP 400 `{error: "Validation failed", fields: [...]}`. Client-side is UX; server-side is security.
- **BOLA** — every resource-by-ID endpoint verifies permission per the documented authorization model:
  - `owner-only`: `if (resource.userId !== session.userId) return 403`
  - `team-scoped`: `if (resource.teamId !== session.teamId) return 403`
  - `role-based`: `if (!session.roles.includes(requiredRole)) return 403`

### Database (applies if project uses a relational database)
- **ID column types** — all PKs and FKs use `BIGINT` (not `INT`, `INTEGER`, `INT32`, or bare `serial`). INT32 overflows at ~2.1B rows — every table, every phase.
- **Migration tooling** — schema changes via migration files only; never raw `ALTER TABLE` in production.
- **Database URL in env var only** — `DATABASE_URL` in `.env.example`. Never hardcoded.
- **Connection pooling** — explicit min/max limits; never open a new connection per request. Platform-specific: Vercel → PgBouncer/Neon; Railway/Render → standard pool 5–20.

### API conventions (applies if project exposes an HTTP API)
- **Route versioning** — all routes under `/api/v1/` from Phase 0. Cheap to add early; expensive to retrofit.
- **Pagination on all list endpoints** — `limit` (max 100, default 20) + `cursor`/`offset`. Never unbounded. Default cap enforced server-side.
- **Consistent error shape** — `{error: string, code?: string, fields?: string[]}` + matching HTTP status: 400/401/403/404/429/500.

### Health check
- `GET /api/health` — returns `{status: "ok", ts: <ISO>}` with HTTP 200. No auth. Used by deploy platforms for rollback triggers. **On projects where `observability` is also active, `/build-spec` removes the health check line within S6 and replaces it with O4 (deep check with DB ping) during injection. Only one health check appears in `validation.md`.**

### Test auth seed (required for S3 checks)
Two test users — verifies user A cannot access user B's data. Without this, S3 checks cannot run.
- `testuser-a@test.local` / `testuser-b@test.local` with known passwords
- Helper that logs in each and returns session cookies/tokens (for curl/test scripts)
- Gated behind `NODE_ENV=test` or `--seed-test` — never runs in production

---

## Tier 2 — Ask Once (`/build-spec` constitution mode)

Asked during `/build-spec` (constitution mode). Answers stored in `tech-stack.md ## Safety Defaults`. Not re-asked in subsequent phases. Must be written verbatim to `tech-stack.md ## Safety Defaults` by `/build-spec` (constitution mode).

| Key | Question | Options | What gets built |
|---|---|---|---|
| `production_domain` | What's your production URL? | Free text OR "TBD" | CORS pre-configured with real domain; if TBD, uses `CORS_ORIGIN` env var and S8 blocks until set |
| `billing` | Will this app charge users? | Yes / Planned for later / No | Yes/Planned → Stripe + webhook `constructEvent()` signature check + idempotency key handling scaffolded |
| `sensitive_data` | Does this app handle money, health records, or legal documents? | Yes / No | Yes → append-only `audit_log` table + middleware scaffolded. Use outcome-only framing — do not say "audit log" to the user. |
| `error_tracking` | When something breaks in production, how do you want to be alerted? | Sentry free tier (recommended) / Logs only | Sentry → SDK + `SENTRY_DSN` in `.env.example`; Logs only → structured JSON logging |
| `deploy_platform` | Where will this deploy? | Vercel / Railway / Render / Other | Platform-specific README setup, env var workflow, health check wiring |

`error_tracking` (SD4) is also read by `observability-baseline.md` — no re-ask needed there.

---

## Tier 3 — Safety Validation Block (injected into `validation.md`)

Inject verbatim for the first phase. Subsequent phases inherit S1/S2/S5/S6/S7/S8; add only new endpoint rows to S3/S4. When `observability` is also active, `/build-spec` replaces the S6 health check line with O4 (pagination check in S6 remains).

Tags: `[static]` = file/code check; `[runtime]` = booted dev server required; `[deploy-time]` = only verifiable post-deploy (checked at dogfood handoff). All non-`[deploy-time]` checks are hard blocks in Round 1.

```markdown
## Safety Baseline Checks

### S1 — Secret hygiene [static]
- [ ] `.env` not tracked: `git ls-files .env` returns empty
- [ ] `.env.example` exists with all keys documented
- [ ] CI secret scanning step present in `.github/workflows/` (or platform equivalent)

### S2 — Security middleware [runtime]
- [ ] Security headers present: `curl -I <health-endpoint>` shows `X-Frame-Options`, `X-Content-Type-Options` headers (HSTS omitted in dev; it's prod-only)
- [ ] CORS — positive case: `curl -H "Origin: https://<prod-domain>" <api-endpoint>` returns `Access-Control-Allow-Origin: https://<prod-domain>`
- [ ] CORS — negative case: `curl -H "Origin: https://evil.com" <api-endpoint>` does NOT return `Access-Control-Allow-Origin: https://evil.com`
- [ ] Rate limiting on auth: 11 rapid POST requests to auth/login endpoint, 11th returns HTTP 429
- [ ] `trust proxy` configured correctly if behind CDN/proxy (check `X-Forwarded-For` is trusted)

### S3 — Authorization checks [runtime]
Requires a two-user seed fixture (user A and user B with separate session tokens) — must be created as part of the safety scaffold.
For each resource endpoint in this phase (fill in actual paths):
- [ ] `GET /api/v1/[resource]/:id` — user B's token requesting user A's resource → HTTP 403 (not 200, not 404)
- [ ] `PATCH /api/v1/[resource]/:id` — same → HTTP 403
- [ ] `DELETE /api/v1/[resource]/:id` — same → HTTP 403

### S4 — Input validation [runtime]
For each mutation endpoint in this phase:
- [ ] Empty body → HTTP 400 with `{error: "Validation failed"}` (not 500)
- [ ] Wrong field types → HTTP 400 (e.g. string where number expected)
- [ ] Oversized field → HTTP 400 (e.g. name field >255 chars)

### S5 — Database types [static]
- [ ] No dangerous ID types on id/foreign key columns. Run the appropriate check for the ORM in use:
  - **Raw SQL migrations:** `grep -rin '\bid\b.*\b(serial|integer|int4)\b\|\b(serial|integer|int4)\b.*\bid\b' migrations/ --include="*.sql" | grep -vi bigserial` returns empty
  - **Prisma:** `grep -n '^\s*id\s\+Int\b\|@id.*Int\b' prisma/schema.prisma` returns empty
  - **Drizzle:** `grep -rn 'integer(.*id\|id.*integer' src/db/ --include="*.ts"` returns empty
  - Safe types: `BIGINT`, `BIGSERIAL`, `bigint()`, `bigserial()`. Any match above is a blocking failure.
- [ ] No raw `ALTER TABLE` in source: `grep -r "ALTER TABLE" src/` returns empty

### S6 — API conventions [runtime]
- [ ] List endpoints cap unbounded requests: `GET /api/v1/[collection]` (no limit param) returns ≤ 20 items, not the full table
- [ ] Health endpoint: `curl <host>/api/health` → HTTP 200, JSON with `status` and `ts` fields

### S7 — Auth hygiene [static]
- [ ] Password hashing library (bcrypt/argon2) is imported and explicitly called in the user-create and user-update handlers — not just imported. Read the handler code; don't use grep.
- [ ] Session/JWT secret loaded from env var: `grep -rn "JWT_SECRET\|SESSION_SECRET" src/` returns at least one match, AND `grep -rn "= ['\"]" src/` doesn't show the secret hardcoded as a string literal
- [ ] Secure cookie flags configured: search the session/cookie setup for `httpOnly`, `secure`, and `sameSite` — all three must be present

### S8 — Pre-deploy environment [deploy-time]
- [ ] All keys in `.env.example` are set in the deployment platform's env config
- [ ] `CORS_ORIGIN` set to actual production domain (not localhost, not `*`)
- [ ] If Sentry chosen: `SENTRY_DSN` is set and a test event is visible in the Sentry dashboard
- [ ] DB connection verified in production environment (via health endpoint DB check from observability-baseline)
```

---

## What this does NOT cover

- **Full CI/CD pipeline** — preview environments and rollback are platform-handled (Vercel, Railway, Render)
- **Advanced observability** — structured logging, uptime monitoring, and enhanced health checks live in `observability-baseline.md`
- **Code quality tooling** — linting, formatting, pre-commit hooks in `dx-baseline.md`
- **Repository management** — README structure, PR templates, branch strategy in `repo-baseline.md`
- **Accessibility** — `build-design/references/app-shell-spec.md` (+ `build-design/references/ux-rules.md` §3)
- **Performance budgets** — post-PMF; address when real traffic data exists
