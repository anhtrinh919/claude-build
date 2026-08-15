---
name: build
description: SDD build orchestrator. Turns an idea into a shipped, dogfooded product, one vertical slice per phase. Re-entrant and state-first — reads `.build-state.json` to resume after context loss, then routes /build:shape, /build:spec, /build:design, /build:backend, /build:review. Every step and sub-skill handoff auto-continues in the same turn; only named gates (constitution, phase-complete, a felt-impact fork) stop and wait. Trigger on /build.
---

# /build — SDD orchestrator

**Wiring only.** Every sub-skill owns its own body. These tables say where you are and which sub-skill runs next — never open a sub-skill file just to find that out. **Start at *Cold start*.**

- Re-entrant and state-file-first. Every gate writes `.build-state.json`. Any turn can be a cold resume.
- **Sequential:** shape, then per phase spec → design → backend → review. No background or parallel builds.
- The orchestrator **is the session**. Every sub-skill loads into it through the Skill tool and runs **inline** — same session, never spawned as a subagent.
- Quoted user-facing text is the *intent* to convey, never a script. Never show the user a path, a stack name, or jargon (`_shared/voice.md`).

**One turn per phase.** Internal step and sub-skill-return boundaries are not stopping points — cross them silently, same turn. Stop and yield ONLY at the named user gates: the constitution boundary, each phase-complete boundary, the Outcome Card gate, a felt-impact fork, a cap-hit binary, the external design hand-off. No "next I'll…", no permission-to-proceed, no pause narration. Full contract: `_shared/auto-continue.md`.

## Invocation contract

| Mode | Model | Mechanism | Reads | Writes | Terminal step |
|---|---|---|---|---|---|
| single | Opus | inline — loaded via the Skill tool into this session, never spawned as a subagent | `.build-state.json`, `product.md` | `.build-state.json` at every gate | `constitution-complete`, `design-complete`, `backend-complete`, `roadmap-complete`, every rollback |

Off-pipeline skills this orchestrator does not route: `/build:polish` (backlog drainer), `/build:migrate` (legacy upgrade). The user invokes both directly. **Neither advances `step`**, so an open phase stays open across a whole polish run — each reads `.build-state.json` and names the outstanding pipeline step, on entry and again at wrap.

## Model + mechanism
**Every sub-skill is a separate, flat, top-level folder** in `skills/` — `${CLAUDE_PLUGIN_ROOT}/skills/spec/`, never nested under `build/`. Invoke each by name through the Skill tool (`skill: "build:spec"`). Only `schemas/` and `_shared/` sit under `build/`; never guess a path from those.

Each sub-skill runs inline and lists its own leaf workers. The rule that generates them: **authoring or deciding → Opus; executing → Sonnet.** Core docs are Opus-authored. **Ponytail full** governs all implementation work, never core-doc authoring, which is judged on completeness.

## State schema — `.build-state.json`
One owner per field. Separate fields prevent write races.

| Field | Values | Owner |
|---|---|---|
| `stack` | `"build"` — which orchestrator owns this project. Set at the first state write, carried forever. Absent → the orchestrator you are in backfills it. Never guess it from step names | orchestrator |
| `phase` | number (0 = Foundation) | orchestrator |
| `feature` | slug | orchestrator |
| `step` | see Resume ladder | see *Who writes `step`* |
| `reviewIteration` | fix-loop counter, cap 3 | /build:review |
| `requirementsHash` | sha256 of requirements.md (drift detector) | /build:spec (phase) |
| `currentSubStep` | breadcrumb, e.g. `"design.phase-1"`, `"shape.1.2"`, `"spec.1.5"`. Null on clean exit | whichever step runs |
| `dogfoodPid` | running dogfood server PID. Null on stop | /build:review |
| `phaseCeremony` | `"full"` \| `"narrow"` — set per phase at Outcome Card approval | /build:spec (phase) |

There is no `foundationStatus` — v2 has no background foundation build.

**Breadcrumbs.** `currentSubStep` takes one value per literal step: `"shape.1.1"`–`"shape.1.2"` in /build:shape, `"spec.1.4"`–`"spec.1.6"` in /build:spec constitution mode. A resume mid-step continues the interview at that step. It never restarts the tree.

**Legacy state.** A `foundationStatus` field, a `frontend-*` step, an `*-approved` step, or a `.build2-state.json` means the project predates this stack. Stop and run `/build:migrate` — that skill owns detection and upgrade.

## Resume ladder
The chain of `step` values. `‖` marks a **boundary gate**: `/eli` wrap, then an `AskUserQuestion` (Proceed / Stop-for-now), then yield the turn. Every other arrow **auto-continues in the same turn** — no stop, no phase-wrap ("Phase N is built"), no "paused" framing, and never tell the user to retype `/build`.

`_shared/auto-continue.md` is the full list of real stops — this diagram shows only its two named boundaries, `‖`. Landing on a non-`‖`, non-terminal row without having run its named sub-skill is a routing failure.

```
(no state) → shaping-in-progress → shape-complete → constitution-complete ‖
  → spec-complete → design-complete → backend-complete → phase-complete ‖
  → replan → next phase, back to spec-complete
           → no phase left → roadmap-complete (end)
```

Detail, where a step is more than its next link:

| `step` | Resume at |
|---|---|
| *(no state, no mission.md)* | /build:shape 1.1 concept interview → 1.2 3C research. Write `stack: "build"` + `shaping-in-progress` + `currentSubStep: "shape.1.1"` before the first question |
| `shaping-in-progress` | /build:shape at `currentSubStep`. Continue there, never restart the tree |
| `shape-complete` | /build:spec constitution: 1.4 product interview → 1.5 constitution writing → 1.6 roadmap. Resume at `currentSubStep` if one is set |
| *(no state, mission.md present)* | /build:spec **replan** → feature cycle |
| `constitution-complete` | Feature cycle → Phase 0 spec |
| `spec-complete` | /build:design — or straight to /build:backend when `phaseCeremony: "narrow"` |
| `design-complete` | /build:backend |
| `backend-complete` | /build:review (silent handoff). No roadmap phase left after this one → pass `final-phase` so its dogfood runs unfenced across the whole product |
| `phase-complete` | check `dogfoodPid` → dogfood handoff if needed → boundary gate → (Proceed) /build:spec replan **always** → a roadmap phase left → its spec; none left → write `roadmap-complete` and stop |
| `roadmap-complete` | **Terminal.** Every roadmap phase is built. "Nothing left to build." Stop. |
| `phase-blocked` | surface the open issues. Never auto-resume. Always stops |

**A `step` write is a trigger, not a milestone.** Every non-`‖` row in the ladder writes `step` and then keeps working. So the message that writes one of those values **also carries the next sub-skill's invocation** — bookkeeping and the next move ride together, and the report comes after the work rather than after the write. Ending a message on a `step` write is the auto-continue failure, whatever the message says.

This is the checkable form of the rule stated above, and it exists because the prose form has failed four times: reported after `3.1.1`, after `4.0.1`, after `4.0.2`'s repetition restore, and again on a `spec-complete` → `/build:backend` hand-off that ended with "starting Group 1" and stopped. **A long report reads like the end of a turn**, and that pull beats a prohibition. It does not beat a rule about where a tool call has to sit.

**Who writes `step`.** A sub-skill writes it on clean exit when no gate follows: `shaping-in-progress`/`shape-complete` (/build:shape), `spec-complete` (/build:spec phase), `phase-complete`/`phase-blocked` (/build:review). The **orchestrator** writes every gated step after its gate passes — `constitution-complete`, `design-complete`, `backend-complete`, `roadmap-complete` — and every rollback. /build:design, /build:backend, and /build:spec constitution mode return without writing `step`: it rides on the previous terminal step while `currentSubStep` tracks live position, cleared to null on their clean return.

**Roadmap discipline** (`_shared/roadmap-axis.md`). Phase 0 = Foundation: shell + hero screens, polished static, unwired. Phases 1+ = vertical slices, slice-tested, never horizontal. /build:spec drafts the sequence; the user confirms it.

## Transition gates
After a sub-skill returns, run its gate before you write its terminal `step`. Pass → write `step`, auto-continue. Fail → roll `step` back and re-run the skill. Loop until pass. These never stop for the user.

| Gate | Pass condition | On fail |
|---|---|---|
| **design-compliance** | every screen and state in requirements.md has a design artifact — `claude-code`: a mockup in `specs/<phase>/mockups/`; `external`: an image in the `handover.md` index | rollback `step`→`spec-complete`, re-run /build:design |
| **backend-compliance** | integration test green for every API contract in requirements.md | rollback `step`→`design-complete`, re-run /build:backend |

## Handoff contracts
Quick map only. Each sub-skill's own `## Invocation contract` is canonical. Doc schemas: `schemas/`.

| Sub-skill | Reads | Writes |
|---|---|---|
| /build:shape | the idea | **Product Shape** (prose, no file) + `research.md` |
| /build:spec · constitution | Product Shape, `research.md` | `mission.md` `product.md` `tech-stack.md` `roadmap.md` + living-docs scaffold |
| /build:spec · phase | constitution, `roadmap.md`, `backlog.md` | approved `outcome-card.md` + `specs/YYYY-MM-DD-[feature]/{requirements,plan,validation}.md` + `requirementsHash` |
| /build:spec · replan | the just-completed phase | living docs updated, changelog, branch merged |
| /build:design | `requirements.md` | `design-tokens.css` + (`claude-code`: `mockups/`, non-visual notes→`design-notes.md`, **no handover**) / (`external`: `design-brief.md` + images + `design-comment.md` + `handover.md` index) |
| /build:backend | `requirements.md` `plan.md` `design-tokens.css` + the mode's design source | working, integration-tested API |
| /build:review | `validation.md` `outcome-card.md`, running app | review report + silent fixes + **dogfood handoff** |

**Contracts that never move.** `outcome-card.md` is the user's contract, frozen on approval — a card change restarts /build:spec phase mode. `requirements.md` is the machine contract shared by design and backend, hashed, within-phase. Cadence for every other doc: `schemas/living-docs.md`.

## Narrow phase (`phaseCeremony: "narrow"`)
Set by /build:spec at Outcome Card approval, only for a phase touching screens an earlier phase already built.

/build:spec runs without drafters, reconciliation, or drift review, and writes no `validation.md`. **/build:design and design-compliance are skipped entirely** — `spec-complete` goes straight to /build:backend, unchanged. /build:review runs **`standalone-dogfood`** mode and, as this phase's named closure, still writes `phase-complete`/`phase-blocked`.

## Cold start
Once per session, first action. Read both files in the contract → find `step` in the Resume ladder → resume there. Skip `product.md` if missing. Not build orchestration → ignore the state file; it is an anchor, not a coercion.

**Stall check, before you resume.** A phase whose code is committed but whose `step` never advanced looks exactly like a phase that never started. Two signals say otherwise, and either one means the work is done and only a gate is outstanding — run that gate, do not rebuild:
- `currentSubStep` reads `backend.done` — /build:backend finished and handed back.
- `step` is `spec-complete` or `design-complete`, and every `specs/<phase>/verify-group-*.sh` passes.

**Never tell the user a phase is built.** A phase is `phase-complete` or it is unfinished, and the words for the middle state are **"built, not closed — `/build:review` still has to run."** "Phase N is built" reads as done, and a user who hears it reasonably stops the pipeline and goes elsewhere. Say which step remains, by name, every time you report progress on an open phase.

**One door** (`_shared/entry-point.md`). A resume routes through here, never straight into a sub-skill. The `stack` field names the owning orchestrator.

**One turn per phase.** Internal step and milestone boundaries are not stopping points — cross them silently and keep working. Stop and yield the turn ONLY at the named user gates, per the Resume ladder above and `_shared/auto-continue.md`.

## Ground rules (canon in `_shared/`)
1. **/eli only at the user-gate boundaries** — the wrap before a constitution or phase-complete go/no-go. A summary at an internal step reads as a stop. Cross every other internal step and sub-skill return silently, same turn — no "next I'll…", no permission-to-proceed (`_shared/auto-continue.md`).
2. **User gates are outcome-only; every felt decision is a fork** (`_shared/voice.md`). The user approves the shape, the product story, the Outcome Card, the design, the phase-end dogfood — never spec files. Any decision with felt UX or performance impact becomes a fork: options, a plain tradeoff each, a recommended default. Front-load it; a fork emerging mid-build overrides auto-continue. Invisible plumbing → decide silently → the doc that owns it (`tech-stack.md` or `docs/architecture.md`).
3. **Backlog capture.** A deferred request → a dated one-liner in `backlog.md` at once, confirmed by ID ("Noted as T-7"). Dogfood bugs thread as `DF-N`. Always refer by ID.
4. **Agent containment** (`_shared/subagent-policy.md` is canon). Every leaf brief ends with its containment string, verbatim. After every return check `git log <pre>..HEAD` and `ps ax | grep "[c]laude -p"` for runaways. **Agents never commit** — the orchestrator does.
5. All user questions go through `AskUserQuestion`. Cap-hit and blocker surfaces are binary: "Accept anyway, or Stop?"
6. **Pivoting.** Mid-phase is rule 2's fork. Post-ship is replan's changelog entry. Mission-outgrown re-runs only the affected constitution slice (1.4 mission, 1.5 tech constraints), never all of Milestone 1.
