---
name: build-deploy
description: >
  SDD Milestone 3 (Deploy) — the closing skill for /build, replacing the old "roadmap-complete → stop"
  dead end. Four steps: 3.1 whole-codebase review (reuses the installed `ponytail-review` skill against
  the full-project diff — advisory, never blocks); 3.2 whole-app blind dogfood (reuses the `dogfood`
  agent, unscoped from any single phase, walking every Named Flow in mission.md's Master User Journey);
  3.3 commit/merge verification (a no-op check — every phase already merged incrementally via
  `build-spec replan`, this just confirms nothing's left dangling); 3.4 deploy — real automation on
  Vercel via the `deploy_to_vercel` MCP tool, a guided handoff for Railway/Render/Other. Runs inline.
  Invoked by /build at `roadmap-complete`; also user-invocable standalone for re-deploys.
user-invocable: true
argument-hint: "(optional — re-deploy an already-shipped project)"
---

> **Part of `/build`.** On a resume with an active `.build-state.json`, enter through the `/build` orchestrator — it routes here; don't drive this skill off the state file directly (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/entry-point.md`, which also lists the standalone-invocable modes).

# /build-deploy — Milestone 3: whole-app review, dogfood, deploy

Where a quoted string is user-facing, it's the *intent* to convey in plain language, not a script to recite — never file paths, stack names, or jargon (`${CLAUDE_PLUGIN_ROOT}/skills/build/_shared/voice.md`).

## Invocation contract

| Step | Reads | Writes | Terminal step |
|---|---|---|---|
| 3.1–3.4 | `roadmap.md` (complete), `tech-stack.md ## Choices` (Hosting / Deployment), `product.md`, `mission.md ## Master User Journey`, every `specs/*/outcome-card.md` | optional cleanup commits, README status line, optional deploy config, `.build-state.json` | **`deploy-complete`** on a clean run, **`deploy-blocked`** on an unresolved Stop — written on clean exit |

Model: Opus main. Leaves: none for 3.1 (ponytail-review runs inline), `subagent_type: dogfood` for 3.2 (Sonnet), Sonnet fix leaves for either round's fix loop. Mechanism: inline (holds gates, dispatches leaves — same as every other sub-skill; `_shared/subagent-policy.md` Rule 1).

**Mode detection:** invoked by /build when `.build-state.json` `step: "roadmap-complete"` — start at 3.1. Invoked directly by the user on an already-deployed project — read `deploy-complete`'s prior state, treat as a re-deploy: skip straight to 3.4 unless the user asks for a full re-run.

---

## Step 3.1 — Whole-codebase review

Write `currentSubStep: "deploy.3.1"`. Compute the full-project diff:

```
git diff $(git rev-list --max-parents=0 HEAD)..HEAD
```

No root history reachable (shallow clone) → fall back to `git diff --stat HEAD` and note the limitation in the report. This is deliberately **not** a correctness re-review — that already ran per-phase in build-review; re-running it here would be waste. It's an over-engineering/bloat sweep across everything that accumulated phase by phase.

Invoke the `ponytail-review` skill over that diff — reuse it directly, don't rebuild review logic. **Advisory only, never blocks.**

- Zero findings → convey "Lean already. Ship." and move to 3.2.
- Findings → present the tagged list (`delete:`/`stdlib:`/`native:`/`yagni:`/`shrink:` + a one-line net-effect summary), one `AskUserQuestion`: **apply some** / **apply all** / **skip**.
- Applied fixes → Sonnet edit leaves, one commit each: `simplify: <tag> <one-line>`.

## Step 3.2 — Whole-app blind dogfood

Write `currentSubStep: "deploy.3.2"`. **Dispatch `subagent_type: dogfood` directly** — a sub-skill can't invoke a sibling sub-skill, so this does not route through build-review.

Briefing (same blindness discipline as every other dogfood dispatch — never the diff, never how anything works): the `mission.md` one-line purpose as the goal, URL + credentials, **no scope fence** — the whole product is in scope, not one phase. For its Phase 2 (guided walk), supply every Named Flow from `mission.md ## Master User Journey` (Layer 2) and every primary outcome from every `specs/*/outcome-card.md` found on disk, oldest phase first.

**Reading the return + fix loop.** Same shape as build-review's fix loop: HIGH and MEDIUM findings → silent auto-fix, wave-dispatched Sonnet leaves, root-cause tag (`[root-cause]`/`[symptom-patch]`/`[heuristic]`, the last two need explicit user opt-in), targeted re-verify, cap 3. LOW → fix if trivial, else `backlog.md` as `T-N [polish]`. Cap-hit or a felt-impact tradeoff surfaces the same binary as everywhere else: **Accept anyway, or Stop?** Stop → write `{ step: "deploy-blocked", currentSubStep: null }`, halt here — never auto-resume.

## Step 3.3 — Commit/merge verification

Write `currentSubStep: "deploy.3.3"`. Mostly a no-op — every phase already merged its own branch to `main` incrementally via `build-spec replan`. Checks only, no new work unless one fails:

- `git status --porcelain` on `main` must be clean.
- `git branch --list 'phase-*'` — any unmerged phase branch is a red flag from a missed replan step. Surface it, offer to merge or delete (confirm with the user) — never silently merge.
- Every `roadmap.md` phase has a `built` row in `product.md`'s Screen Inventory (not `removed`/still-planned) — a mismatch surfaces, doesn't silently proceed.
- `CHANGELOG.md` has an entry for every phase (cheap grep sanity check).

## Step 3.4 — Deploy

Write `currentSubStep: "deploy.3.4"`. Read the hosting/deployment target from `tech-stack.md ## Choices`. `TBD`/absent/unspecific → ask once via `AskUserQuestion` (Vercel / Railway / Render / Other), then patch the `tech-stack.md ## Choices` **Hosting / Deployment** line with the answer so a re-deploy doesn't ask again.

**Vercel — real automation.** Confirm every var in `.env.example` is actually set (the `vercel:env` skill, or the `vercel` MCP tools) — prompt the user for anything missing, never invent a secret value. Run `deploy_to_vercel` (MCP tool). Poll `get_deployment` / `get_deployment_build_logs`. Build failure → pull `get_runtime_errors`, attempt at most one root-cause fix + redeploy; a second failure hands off with the logs rather than looping.

**Railway / Render — guided handoff, honest about the ceiling.** No CLI automation path in this environment. Prepare what's needed: confirm build/start commands, list required env vars, write/verify `docs/deployment.md`. Then give the user a plain checklist: create the project on the platform's dashboard, connect the repo, set these env vars, deploy. If a CLI is present on `$PATH` **and** already authenticated (an existing token — never attempt an interactive OAuth login on the user's behalf), offer to run it as a convenience; otherwise it's the user's own step.

**Other — fully guided.** Write the generic `docs/deployment.md` (env vars, build command, start command, health-check endpoint if the app exposes one) and stop there.

Either way: update `README.md`'s status line ("Deployed — `<URL>`" or "Ready to deploy — see `docs/deployment.md`"), write `{ step: "deploy-complete", currentSubStep: null }`.

---

## Ground rules

- **This is not a second code review.** 3.1 hunts bloat only (ponytail lenses); correctness was already gated per-phase by build-review. Don't re-litigate what already passed.
- **3.2 is genuinely unscoped** — the one place in the whole stack where dogfood is deliberately NOT phase-scope-fenced. If it finds something that "isn't this phase's problem," that's the point: nothing upstream ever looked at the whole product at once.
- **Never invent a secret.** Any missing env var for 3.4 is a question to the user, never a placeholder value written to a real deploy target.
- **Agents never commit**, and 3.4 never pushes to a platform without the user having confirmed the platform choice at least once (the tech-stack hosting choice, or this step's own ask). The orchestrator/this skill owns commits; leaf briefs carry the containment line — no spawning agents, no `/build*`, no `claude -p`, no server, don't address the user.
- **All user questions via `AskUserQuestion`.** Cap-hit / blocker surfaces are binary: "Accept anyway, or Stop?"
