Two drafter briefs, one envelope. Spawn one Opus leaf per task, in parallel. Give each leaf the shared block below plus its own task block — never both task blocks.

## Shared block (goes to both leaves)

Do not write any files — return content only.

Drill decisions (verbatim from the /build:spec phase drilling session): [paste full phase decisions verbatim]
Outcome card (the user-approved contract — everything you write must serve it): [paste outcome-card.md verbatim]
Scope summary: [paste scope challenge output — what already exists / what's new]
Phase: [N], Feature slug: [slug]
Tech stack: [paste tech-stack.md — constraints, non-negotiables, pinned versions, key technical decisions]

End your return with a `## Latent decisions` list — one sentence per choice you made that was not in the drill decisions or the outcome card. For any latent decision the user would feel (UX or performance), carry its fork: the genuine options, a one-line plain-language tradeoff for each, and your recommended pick. No other commentary.

Close every brief with the containment string from `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/subagent-policy.md`.

## Task A — requirements.md + validation.md

Add to the shared block: `Spec directory: specs/YYYY-MM-DD-[slug]/`

Write requirements.md using the schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/requirements.md`. Fill every section:
- Frontmatter: phase (integer), ui (true | false)
- Scope: one paragraph on what this phase delivers
- User stories: one per major user action, each referencing its Named Flow + step from product.md, format [Flow name, Step N]
- UI requirements: every screen and every state (default, empty, loading, error, mobile) — behavior and key elements only, no visual treatment
- Data model: all tables/schemas, field names, types, relationships
- API contracts: every endpoint — method, path, **consumed-by screen(s)** (name them, or `internal` for webhook/cron/server-to-server), auth, request shape, success response, and ALL domain-specific error conditions, not just 500. Every endpoint names a consumer; every screen with data needs has a backing endpoint. **Use PUT for full-resource update endpoints — never PATCH unless the spec explicitly requires partial-update semantics.** **Document the JWT signing algorithm (HS256), token expiry, and that the JWT secret comes from environment variable `JWT_SECRET`.**
- Implicit states (mandatory, every phase): (a) optional fields — state valid as null or absent; (b) array fields — empty `[]` is valid, not just null; (c) whitespace-only required strings — invalid, identical to empty (trim before the empty check); (d) GET list endpoints — return `200 + { items: [], total: 0 }` on zero records, never `404`; (e) any configurable limit (`pageSize`, tags count) — state the maximum as a number; (f) string-based filter params (tag, category, status) — state whether matching is case-sensitive; (g) invalid pagination params (`page=0`, `page=-1`, `pageSize=0`) — clamp to a valid floor (page→1, pageSize→1) or return 422.
- Constraints & context: business rules, tone, patterns from tech-stack.md
- Excluded from this phase: named explicitly

Write validation.md using the schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/validation.md`:
- Automated checks: specific commands that exit 0 on success (typecheck, named unit tests, one curl per API contract)
- Manual verification: specific steps at named viewport sizes, binary pass/fail. Every user story gets at least one
- Outcome checks: one binary, demonstrable check per outcome-card primary outcome, phrased so a non-technical person can verify it on screen
- Definition of Done: all criteria, with the outcome checks part of it

Return both files as raw markdown, each labelled with its filename.

## Task B — plan.md

Add to the shared block: `Existing codebase relevant context: [paste key file paths, route structure, component names for files this phase touches]`

Write plan.md using the schema at `${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/plan.md`. Organize as numbered task groups, each independently verifiable:
- Describe what each group delivers — user-visible capability, data path, API surface, integration point — naming components, files, modules, and endpoints specifically
- Each group carries a one-line Verify command and a Depends-on field
- Design-agnostic: no hex values, utility classes, pixel sizes, or font names — those come from the design source at build time
- Sub-tasks specific enough to implement without ambiguity, each group implementable and verifiable before the next starts

Return plan.md as raw markdown.
