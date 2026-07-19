# Subagent Policy — shared by every SDD skill

The rules for spawning, briefing, and trusting subagents anywhere in the build stack. Skills reference this file instead of restating the rules. If a skill's text ever contradicts this file, this file wins — fix the skill.

## Rule 1 — One level of nesting only

**Subagents cannot spawn subagents.** The Agent tool is not available inside a subagent — an inner spawn silently degrades into the subagent doing the work itself, defeating the design (blindness, parallelism, model split all break).

Consequence for skill design:
- **In `/build`, every sub-skill runs `inline`.** The orchestrator (`/build`) is the session; it loads and runs each sub-skill in turn and never spawns one as a subagent. The only subagents anywhere are the leaf workers a sub-skill fans out (research, drafters, render, browse, implementation, fix) — and those are always internally subagent-free.
- Any leaf brief that says "read and execute a SKILL.md" is a bug: it risks a second-level spawn that silently degrades. Brief a leaf with a concrete task + file list, never a skill to run.
- Before giving any leaf worker orchestration duties, re-check this rule.

## Rule 2 — Two briefing styles; pick deliberately

| Style | Shape | When |
|---|---|---|
| **Skill-loader** | `Read and execute ~/.claude/skills/<x>/SKILL.md per its Invocation contract. <slot-filled inputs>` | The subagent runs a whole pipeline and full skill context is wanted. |
| **Context-isolated inline brief** | Self-contained prompt; only the named slots are passed; relevant doc content pasted in | Blindness or scope-limiting is the point: naive reviewers, skeptic panels, drafters, fix agents. Never pass the conversation, the diff, or "what was just built" unless the brief explicitly calls for it. |

The blind-reviewer hard rule (see the `dogfood` agent's Phase 1) is the strictest instance of the second style — what the caller must NOT brief is part of the contract.

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
- **Agents never commit.** They return: changed files, verify results, `status: done | blocked | needs-decision` + diagnosis. The orchestrator integrates, runs cross-agent interface checks, and commits per group.
- **Agents never start dev servers** (port collisions) and **never address the user**. Any "surface to user" / "stop and ask" rule inside an agent's discipline translates to: return `status: blocked` + diagnosis; the orchestrator decides whether to retry (fresh agent) or surface.
- **Felt-impact fork mid-build → `status: needs-decision`, never a silent pick.** If an agent hits a decision the user will *feel* — a UX or performance choice with no strictly-better option, not already settled in the spec — it returns `status: needs-decision` + the fork (the genuine options, each option's one-line plain-language tradeoff, a recommended default) and stops that group. It does NOT choose and proceed. The orchestrator surfaces the fork to the user (`AskUserQuestion`, plain language) and re-dispatches a fresh agent with the answer. Reserve this for *real* forks — invisible plumbing the agent decides itself and records in `docs/decisions.md` (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).
- Escalation: if non-overlapping file sets are genuinely impossible for a phase, use worktree isolation (`isolation: "worktree"`) — expensive, last resort.

## Rule 7 — Fresh instance per retry

Once a subagent has seen a discrepancy or a failed attempt, it is no longer a clean read. Re-verification, repeat skeptic rounds, and fleet re-runs always spawn **fresh** instances with the same brief shape — never continue a contaminated one. (Same rule the `code-reviewer` and `dogfood` agents both depend on for re-verify.)

## Rule 8 — Image hygiene: main session never holds base64 images

The orchestrator (main session) must never contain base64 image data in its context. Any work that produces screenshots, design-frame captures, rendered mockups, or PDF pages must run inside a subagent. The subagent saves output files to `/tmp/` and returns file paths + a text verdict. The main session receives paths only — if it must reference an image for the user, it posts the path and lets the terminal/IDE render it; it never embeds the raw bytes.

Applies to: the design-time visual gates (density / defects / responsive / reference), any "compare" or "inspect" workflows, PDF QA loops. If a skill's current text has the main session capturing or posting screenshots inline, that section violates this rule — fix it before the next phase.
