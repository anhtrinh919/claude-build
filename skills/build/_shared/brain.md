# Brain integration (OPTIONAL) — shared across SDD skills

This wires in an **optional external memory system** — a local "brain" (a learnings database queried through a `bun` CLI). It is pure enrichment: past learnings sharpen a skill's judgment when the system is installed, and its absence changes nothing about how the build runs.

**Presence guard — check first.** If `$HOME/.claude/brain` does not exist (the common case for a fresh install), **skip this entire reference**: do not run the commands below, do not log an error, just continue. Everything here is best-effort and never a gate.

Each skill that loads this reference passes its own `$AGENT` (`spec`, `design`, `backend`, `review`) and its `$TAGS` (derived from `tech-stack.md`, up to 5 — e.g. `nextjs,tailwind,bun`). Constitution mode of `/build-spec` has no `tech-stack.md` yet — pass `$TAGS` empty.

## Read brain (only if present)

Run **before** the skill's first substantive step:

```
cd $HOME/.claude/brain && bun run cli/brain-cli.ts query --prompt "$AGENT $TAGS" --limit 5 2>/dev/null
```

- If output is non-empty: index the titles + one-line hooks under `## Relevant past learnings` in the skill's working context. Do NOT inline full bodies — read on demand if a hit looks useful.
- If output is empty or the CLI fails (non-zero exit): log `No prior $AGENT learnings` and continue silently. Never block.

## Write learning (only if present)

Write at two moments per skill: **friction triggers** (scoped to the skill's failure modes) and the **phase-wrap trigger** (once per skill per phase).

```
cd $HOME/.claude/brain && bun run cli/brain-cli.ts add --auto \
  --title "[title]" \
  --tags "$TAGS" \
  --project "[project basename]" \
  --body "[2-5 sentences: what happened, why it matters, what to do]" \
  --agent $AGENT \
  --trigger [phase-wrap|friction] \
  --phase $N
```

`--auto` makes the CLI silent (exit 0/1 only). The skill is responsible for any user-facing notification — typically none, except `/build` which announces aggregate counts.

## Title convention

Titles are **scoped by phase only**, no agent prefix. Cross-skill dedup hinges on `--tags` overlap and title similarity, so an `--agent backend` "Phase 3: foo" and `--agent review` "Phase 3: foo" should converge if they're about the same learning.

- ✓ `Phase 3: bcrypt salt rounds default mismatch`
- ✗ `Phase 3 backend: bcrypt salt rounds default mismatch` ← old format; agent name is already in `--agent`

## Trigger semantics

- `friction` — fired by the skill when something blocked progress (failed verify, missing handover row, Opus flag, scope drift). One entry per distinct issue.
- `phase-wrap` — fired exactly once per skill per phase, at the end, before completion. Captures the meta-learning ("what would a future $AGENT need to know about this kind of phase").

## What NOT to write

- Do not save a friction entry for the same issue twice in one phase.
- Do not save bodies that are activity logs ("then I did X, then I did Y"). Save the *learning*, not the chronology.
- Do not save anything already in CLAUDE.md or the project constitution — those are higher-priority sources.
