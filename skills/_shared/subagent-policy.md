# Subagent Policy — shared by every SDD skill

The rules for spawning, briefing, and trusting subagents anywhere in the build stack. Skills reference this file instead of restating the rules. If a skill's text ever contradicts this file, this file wins — fix the skill.

## Rule 1 — One level of nesting only

**Subagents cannot spawn subagents.** The Agent tool is not available inside a subagent — an inner spawn silently degrades into the subagent doing the work itself, defeating the design (blindness, parallelism, model split all break).

Consequence for skill design:
- A skill that orchestrates its own subagents MUST run **inline** in the `/build` session (mechanism: `inline` in its Invocation contract). Current inline orchestrators: `/spec` Mode 2, `/review`, `/backend`.
- A skill with mechanism `subagent` MUST be internally subagent-free. Current: `/spec` Modes 1+3, `/frontend`, `/ba` Mode 3.
- Before changing a skill's mechanism either way, re-check this rule.

## Rule 2 — Two briefing styles; pick deliberately

| Style | Shape | When |
|---|---|---|
| **Skill-loader** | `Read and execute ${CLAUDE_PLUGIN_ROOT}/skills/<x>/SKILL.md per its Invocation contract. <slot-filled inputs>` | The subagent runs a whole pipeline and full skill context is wanted. |
| **Context-isolated inline brief** | Self-contained prompt; only the named slots are passed; relevant doc content pasted in | Blindness or scope-limiting is the point: naive reviewers, skeptic panels, drafters, fix agents. Never pass the conversation, the diff, or "what was just built" unless the brief explicitly calls for it. |

The blind-reviewer hard rule (see `browser-review-engine.md` Engine 2) is the strictest instance of the second style — what the caller must NOT brief is part of the contract.

## Rule 3 — Model selection

- Use Agent tool aliases only: `sonnet` / `opus` / `haiku`. **Never version-pinned IDs** (`claude-opus-4-6` etc.) — the tool rejects them and pins go stale. Omit `model` to inherit the session default.
- Execution work (browse passes, comparisons, synthesis into templates, doc drafting from a schema) → `sonnet`.
- Architectural reasoning, adversarial review, code fixes → `opus` or inherit the session default.
- In prose, write "Opus (session default)" — never a version number.

## Rule 4 — Output contract in every brief

- Every brief ends with an explicit return format: "Return X as raw markdown / a list in exactly this format. No commentary."
- Drafter/reviewer/skeptic agents **never write files** unless the brief explicitly says so — the caller owns disk writes.
- Structured findings carry severity tags the caller defines, so aggregation is mechanical.

## Rule 5 — Verify on return

Never trust a subagent's "done" claim. After every return: check the expected outputs exist, or re-run the previously-failing check from scratch. A missing output is surfaced, not papered over.

## Rule 6 — Parallel dispatch (fanning out implementation agents)

When a skill dispatches multiple agents to build in parallel (e.g. `/backend` Stage 2):

- **Non-overlapping file sets** per agent. If two groups must touch the same file, they go to the same agent or different waves.
- **Wave barriers.** Topologically sort task groups by their `Depends on:` fields into waves. Wave N+1 starts only after wave N returns; pass the completed groups' API surface + changed-file lists into the next wave's briefs.
- **Agents never commit.** They return: changed files, verify results, `status: done | blocked` + diagnosis. The orchestrator integrates, runs cross-agent interface checks, and commits per group.
- **Agents never start dev servers** (port collisions) and **never address the user**. Any "surface to user" / "stop and ask" rule inside an agent's discipline translates to: return `status: blocked` + diagnosis; the orchestrator decides whether to retry (fresh agent) or surface.
- Escalation: if non-overlapping file sets are genuinely impossible for a phase, use worktree isolation (`isolation: "worktree"`) — expensive, last resort.

## Rule 7 — Fresh instance per retry

Once a subagent has seen a discrepancy or a failed attempt, it is no longer a clean read. Re-verification, repeat skeptic rounds, and fleet re-runs always spawn **fresh** instances with the same brief shape — never continue a contaminated one. (Same rule as `browser-review-engine.md` "Why fresh subagents".)
