# Subagent Policy — shared by every SDD skill

The rules for spawning, briefing, and trusting subagents anywhere in the build stack. Skills reference this file instead of restating the rules. If a skill's text contradicts this file, this file wins — fix the skill.

## Rule 1 — One level of nesting only

**Subagents cannot spawn subagents.** The Agent tool is not available inside a subagent, so an inner spawn silently degrades into the subagent doing the work itself. Blindness, parallelism, and the model split all break.

**"Runs inline" below is about sub-skill loading only — never read it as a ban on Agent-tool leaf dispatch, which this file governs and a project's own `CLAUDE.md` may separately, standingly permit.**

- **In `/build`, every sub-skill runs `inline`.** The orchestrator is the session. It loads and runs each sub-skill in turn and never spawns one as a subagent. The only subagents anywhere are the leaf workers a sub-skill fans out — research, drafters, render, browse, implementation, fix — and those are always internally subagent-free.
- Any leaf brief that says "read and execute a SKILL.md" is a bug. Brief a leaf with a concrete task and a file list, never a skill to run.

## Rule 2 — Two briefing styles; pick deliberately

| Style | Shape | When |
|---|---|---|
| **Skill-loader** | `Read and execute ~/.claude/skills/<x>/SKILL.md per its Invocation contract. <slot-filled inputs>` | The subagent runs a whole pipeline and full skill context is wanted. |
| **Context-isolated inline brief** | Self-contained prompt; only the named slots are passed; relevant doc content pasted in | Blindness or scope-limiting is the point: naive reviewers, skeptic panels, drafters, fix agents. Never pass the conversation, the diff, or "what was just built" unless the brief calls for it. |

For a blind reviewer, what the caller must **not** brief is part of the contract.

## Rule 3 — Model selection

- Use Agent tool aliases only: `sonnet` / `opus` / `haiku`. **Never version-pinned IDs** — the tool rejects them and pins go stale. Omit `model` to inherit the session default.
- Execution work — browse passes, comparisons, synthesis into templates, doc drafting from a schema → `sonnet`.
- Architectural reasoning, adversarial review, code fixes → `opus` or inherit the session default.
- In prose, write "Opus (session default)", never a version number.

## Rule 4 — Output contract in every brief

- Every brief ends with an explicit return format: "Return X as raw markdown / a list in exactly this format. No commentary."
- Drafter, reviewer, and skeptic agents **never write files** unless the brief says so. The caller owns disk writes.
- Structured findings carry severity tags the caller defines, so aggregation is mechanical.

## Rule 5 — Verify on return

Never trust a subagent's "done" claim. After every return, check the expected outputs exist, or re-run the previously-failing check from scratch. Surface a missing output; never paper over it.

## Rule 6 — Capture project directives on the spot

The instant a durable project-scoped rule surfaces, from any skill, save it to the agent's own project memory — not a repo file. Do not wait for replan.

## Rule 6b — Standing rules travel by reference

A project's `CLAUDE.md` auto-loads into every subagent's context regardless of brief content, but nothing about that guarantees a leaf agent weighs it against its task. Any brief whose task could touch a business fact or tune agent behavior adds one line: "Before acting, check this task against CLAUDE.md's binding rules." This does not require re-pasting CLAUDE.md — it cues the model to treat the already-injected project content as active for this task, not background.

## Rule 7 — Parallel dispatch

When a skill fans out multiple agents to build in parallel:

- **Non-overlapping file sets** per agent. Two groups that must touch the same file go to the same agent or to different waves.
- **Wave barriers.** Topologically sort task groups by their `Depends on:` fields into waves. Wave N+1 starts only after wave N returns. Pass the completed groups' API surface and changed-file lists into the next wave's briefs.
- **Agents never commit.** They return changed files, verify results, and `status: done | blocked | needs-decision` with a diagnosis. The orchestrator integrates, runs cross-agent interface checks, and commits per group.
- **Agents never start dev servers** — port collisions — and **never address the user.** Any "surface to the user" rule inside an agent's discipline becomes: return `status: blocked` plus a diagnosis. The orchestrator decides whether to retry with a fresh agent or surface.
- **A felt-impact fork returns `status: needs-decision`, never a silent pick.** The agent returns the fork — the genuine options, each with a one-line plain-language tradeoff, and a recommended default — and stops that group. The orchestrator surfaces it and re-dispatches a fresh agent with the answer. What counts as a felt fork, and where invisible plumbing gets recorded instead: `${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`.
- Escalation: where non-overlapping file sets are genuinely impossible, use worktree isolation (`isolation: "worktree"`). Expensive, last resort.

## Rule 8 — Fresh instance per retry

Once a subagent has seen a discrepancy or a failed attempt, it is no longer a clean read. Re-verification, repeat skeptic rounds, and fleet re-runs always spawn **fresh** instances with the same brief shape. Never continue a contaminated one.

## Rule 9 — Image hygiene: the main session never holds base64 images

The main session must never contain base64 image data. Any work that produces screenshots, design-frame captures, rendered mockups, or PDF pages runs inside a subagent. The subagent saves output files to `/tmp/` and returns file paths plus a text verdict. The main session receives paths only — to show the user an image, it posts the path and lets the terminal render it. It never embeds the raw bytes.

Applies to the design-time visual gates, any compare or inspect workflow, and PDF QA loops.
