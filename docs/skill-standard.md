# Skill standard — claude-build

Every `SKILL.md` in this repo conforms to this. Check a file against it before and after any rewrite.

## 1. Section spine — this order, these names

| # | Section | Required | Holds |
|---|---|---|---|
| 1 | (intro, unnamed) | yes | One paragraph: what this skill is and what it owns. Nothing else. |
| 2 | `## Invocation contract` | yes | The table in §3. Nothing else. |
| 3 | shared concepts | if any | Anything two or more modes/steps below need. One section each. |
| 4 | `## Mode: <name>` or `## Step N — <name>` | yes | The work. See §2. |
| 5 | `## Ground rules` | yes | Numbered. One line each. |

Concept before procedure: a section that defines an artifact, a gate, or a data structure goes above every section that uses it. Never interleave.

Anything that is neither a mode nor a step and is used by only one of them belongs inside that one — not as its own section.

**Orchestrator exception (`build/SKILL.md` only).** It routes; it has no modes or steps of its own. Its slot 4 is the wiring set, in this order: State schema → Resume ladder → Transition gates → Handoff contracts → Narrow phase → Cold start. Slots 1, 2, 3, and 5 apply unchanged. Cold start is the one procedure section and goes last in the set, after every table it reads.

## 2. Vocabulary — one word per concept

| Concept | Use | Never |
|---|---|---|
| A branch — exactly one runs | `## Mode: <name>` | `Track:`, `Case`, `Path` |
| A sequence — all run in order | `## Step N — <name>` | `Stage`, `Round`, `Phase` |
| Mode selection | one row per mode in the contract table | a separate `## Mode detection` section |

A skill with both branches and sequences nests the steps under the mode as `###`.

## 3. Invocation contract — fixed columns

```
| Mode | Model | Mechanism | Reads | Writes | Terminal step |
```

One row per mode. A single-mode skill writes one row named `single`. Every cell is filled; `none` is a valid value, blank is not. `Terminal step` names the `.build-state.json` value this skill writes, or `none`.

Nothing else goes in this section — no voice pointers, no brain triggers, no containment strings. Those live where §4 puts them.

## 4. Canon — cite it, never restate it

`build/_shared/` is the single source. A skill points at the file. A skill never copies the rule inline, not even "for convenience" and not even marked *exact*. A copy guarantees drift the next time the canon changes.

| File | Owns |
|---|---|
| `voice.md` | User is not a developer. Felt decision → fork. Invisible plumbing → decide silently. |
| `subagent-policy.md` | Rules 1–9: nesting, briefing styles, model selection, output contract, verify on return, capture directives, parallel dispatch, fresh instance per retry, image hygiene. **Includes the leaf-containment string.** |
| `entry-point.md` | One door. `stack` routes the resume. The standalone-invocable exception list. |
| `auto-continue.md` | One turn per phase. The named stop list. Return without yielding. |
| `brain.md` | Optional external memory, presence-guarded. |
| `roadmap-axis.md` | Phase 0 = Foundation. Phases 1+ = slices. Slice test. |
| `drilling-discipline.md` | The interview engine. **Generated — never hand-edit.** |

Cite each at most once per file, at the point of use. The intro carries no canon pointers.

## 5. Budgets

| Part | Limit |
|---|---|
| frontmatter `description` | **300 chars**, every file, every time |
| body | a named budget per file |

Verify both with `wc -c`. Never eyeball. Keep literal trigger phrases (`/build:spec`, `"shape this idea"`) verbatim — skill matching needs the exact string, not STE prose.

## 6. Prose rules

ASD-STE100: short sentences, active voice, imperative mood, one word per meaning, no stacked noun clusters, no gerunds-as-nouns. Prose only — never rewrite a code block, command, path, or schema template.

Never hard-wrap. One line per paragraph, bullet, or table row, however long.

Cut: micromanaging instructions, prose that justifies why a rule exists when the rule alone works, retired context, and any reference an earlier cut left dangling. Keep: every mechanism, gate, command, path, and conflict rule.

## 7. Conformance check

Run against a file after any rewrite:

1. Sections appear in §1 order, with §1 names.
2. No `Track:` / `Case` / `Stage` / `Round` headings.
3. Contract table has the §3 columns, one row per mode, no blank cells.
4. No canon rule restated inline — grep the file for text that also appears in `_shared/`.
5. `description` ≤300 chars. Body ≤ its budget.
6. Every `_shared/`, `schemas/`, and `references/` path in the file resolves.
7. Every artifact the skill authors has a schema cited.
