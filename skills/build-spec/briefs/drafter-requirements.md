You are writing the user-facing spec files for a software phase. Do not write any files — return content only.

Drill decisions (verbatim from the /build-spec phase drilling session): [paste full phase decisions verbatim]
Outcome card (the user-approved contract — every requirement must serve it): [paste outcome-card.md verbatim]
Scope summary: [paste scope challenge output — what already exists / what's new]
Phase: [N], Feature slug: [slug], Spec directory: specs/YYYY-MM-DD-[slug]/
Tech stack: [paste tech-stack.md constraints and non-negotiables]

Write requirements.md using schema at ${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/requirements.md. Fill every section:
- Frontmatter block: phase (integer), type (initial | feature | rebuild)
- Scope: one paragraph on what this phase delivers
- User stories: one per major user action, each referencing its Named Flow + step from product.md, format [Flow name, Step N]
- UI requirements: every screen + every state (default, empty, loading, error, mobile) — behavior and key elements only, no visual treatment
- Data model: all tables/schemas, field names, types, relationships
- API contracts: every endpoint — method, path, **consumed-by screen(s)** (name the screen(s) that call it, or `internal` for webhook/cron/server-to-server), auth, request shape, success response, ALL domain-specific error conditions (not just 500). Every endpoint must name a consumer; every screen with data needs must have a backing endpoint. **Verb convention: use PUT for full-resource update endpoints — never PATCH unless the spec explicitly requires partial-update semantics.** **Auth mechanism: document the JWT signing algorithm (HS256), token expiry, and that the JWT secret comes from environment variable `JWT_SECRET` — never hardcode or imply it is configurable per-deployment without naming the env var.**
- Implicit states (mandatory — cover these for every phase): (a) optional fields — explicitly state valid as null or absent; (b) array fields — state that empty `[]` is a valid value, not just null; (c) whitespace-only required string fields — treated as invalid, identical to empty (trim before the empty check); (d) GET list endpoints — return `200 + { items: [], total: 0 }` when the user has zero records, never `404`; (e) any configurable limit (`pageSize`, tags count, etc.) — state the maximum as a specific number so callers know the ceiling; (f) string-based filter params (tag, category, status) — explicitly state whether matching is case-sensitive or case-insensitive; (g) invalid pagination params (`page=0`, `page=-1`, `pageSize=0`) — specify behavior: clamp to a valid floor (page→1, pageSize→1) or return 422.
- Constraints & context: business rules, tone, patterns from tech-stack.md
- Excluded from this phase: named explicitly

Write validation.md using schema at ${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/validation.md:
- Automated checks: specific commands that exit 0 on success (TypeScript typecheck, named unit tests, curl for each API contract)
- Manual verification: specific steps at named viewport sizes, binary pass/fail
- Every user story must have at least one manual check
- Outcome checks: one binary, demonstrable check per outcome-card primary outcome — phrased so a non-technical person can verify it on screen
- Definition of Done: all criteria listed; the outcome checks are part of it

Return both files as raw markdown content, each labelled with its filename, followed by a "## Latent decisions" list — one sentence per choice you made that wasn't in the drill decisions or outcome card (scope inferences, behavior defaults, added exclusions, assumed flows). For any latent decision the user would feel (UX or performance), carry its fork: the genuine options, a one-line plain-language tradeoff for each, and your recommended pick. No other commentary.

Containment: no spawning agents, no /build*, no `claude -p`, no commit, no server, don't address the user — return content only.
