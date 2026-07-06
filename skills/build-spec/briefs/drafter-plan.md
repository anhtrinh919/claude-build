You are writing the technical implementation plan for a software phase. Do not write any files — return content only.

Drill decisions (verbatim from the /build-spec phase drilling session): [paste full phase decisions verbatim]
Outcome card (the user-approved contract — every task group must serve it): [paste outcome-card.md verbatim]
Scope summary: [paste scope challenge output — what already exists / what's new]
Phase: [N], Feature slug: [slug]
Tech stack: [paste tech-stack.md — include constraints, non-negotiables, pinned versions, excluded patterns]
Existing codebase relevant context: [paste key file paths, route structure, component names for files this phase touches]

Write plan.md using schema at ${CLAUDE_PLUGIN_ROOT}/skills/build/schemas/plan.md. Organize as numbered task groups, each independently verifiable:
- Describe what each group delivers (user-visible capability, data path, API surface, integration point)
- Name components, files, modules, endpoints specifically
- Each group: one-line Verify command + Depends on field
- Design-agnostic: no hex values, Tailwind classes, pixel sizes, font names — those come from the design file at build time
- Sub-tasks specific enough to implement without ambiguity
- Typical sequence: primitives/layout shell → page composition → data layer → API routes → integration/edge cases → story walk
- Each group should be implementable and verifiable independently before the next starts

Return plan.md as raw markdown content, followed by a "## Latent decisions" list — one sentence per choice you made that wasn't in the drill decisions or outcome card (tech micro-choices, structural patterns, scoping inferences). For any latent decision the user would feel (UX or performance), carry its fork: the genuine options, a one-line plain-language tradeoff for each, and your recommended pick. No other commentary.

Containment: no spawning agents, no /build*, no `claude -p`, no commit, no server, don't address the user — return content only.
