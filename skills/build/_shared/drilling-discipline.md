<!-- SYNCED FROM ~/.claude/skills/grill-me/references/drilling-discipline.md — do not hand-edit. Re-run sync-drilling-discipline.sh after any change to the canonical file. Source hash: a909b1594c23 -->

# Drilling Discipline

This file is the canonical copy. It applies to grill-me and the build stack's spec skill.

This is a relentless interview, not a checklist. Do not stop at a question count. Stop only when no node is open.

## Four Levels

Every open decision sits at one level: root, trunk, branch, or leaf. Use a level only where a real dependency needs it. Do not add a level to fill the shape.

- Work one thread at a time, depth-first. Finish a branch's leaves before its sibling branch. Finish a trunk's branches before its sibling trunk. Finish a root's trunks before the next root.
- Roots are the exception. Resolve all roots together, in one batch (Step 1).
- A node enters a level only if it passes two gates: the structural gate (below) and the mode's semantic gate (Gate Presets).
- If a node fails a gate: collapse it to the next level down and ask it as a plain question, or move it to the settled list if nothing depends on it.
- Derive each level only from the answer just given to its parent. Do not write a trunk's branches before you have the trunk's answer.

## decision-tree.md

A living document. Add to it as each level resolves; do not write it once upfront. It holds:

- Each root, one sentence, with why it is a root.
- Each trunk under its root, with a one-line scope note.
- Each branch under its trunk, with a one-line scope note.
- Every open leaf, as a question.
- Edges — cross-thread dependencies, in prose.
- Settled items, in a flat table.
- Sequencing — which threads block which.

Record any node you collapsed or rejected, and why.

## Structural Gate — Validity Tests

Run all four tests before you enter a level. If a node fails one test, collapse the level. Do not adjust the question instead.

- **Parent test.** X is a parent of Y only if a different answer to X changes Y's answer, options, or existence. If Y stays the same under any answer to X, they are siblings.
- **Fan-out test.** Each level must be wider than the level above it. Levels of equal width mean the levels were invented first, not derived.
- **Single-child test.** A node with one child is not a branch point. Collapse it into its parent.
- **Settled-fact test.** A decision that nothing depends on is not a root, trunk, or branch. Move it to the settled list.

When real dependencies are rare, keep the tree small: a few roots, each with only its real trunks, each with only its real branches. Record cross-thread dependencies as edges in prose. Do not force a tree shape.

**Inversion check.** Do not leave a child decided while its parent is open. If this happens, mark the child as answered-before-its-parent in the document.

**Random-question check.** If the user says the questions feel random or disconnected, stop. Return to the roots and re-derive the tree. Do not patch the current question.

## Gate Presets, by Mode

The structural gate is the same for every mode. What counts as a root, trunk, branch, or leaf depends on the mode. The call site sets the mode; do not detect it mid-session.

### concept-shape — a new idea

Use for a new project or feature concept.

| Level | Pass | Fail |
|---|---|---|
| Root | The outcome or existential bet — the "why" | Explained by something above it, or already a deliverable |
| Trunk | Names an actor or an unmet need on the path to the root | Already a feature name |
| Branch | A real behavior-level fork, two or more live options | Only one live option |
| Leaf | Concrete and buildable, no open "how" | An open "how" remains |

### phase-scope — a slice of a decided product

Use for the build stack's spec phase mode.

| Level | Pass | Fail |
|---|---|---|
| Root | The phase's Outcome Card outcome | — cite it, do not re-derive |
| Trunk | — | Always fails here. Collapse straight to branch. |
| Branch | An open scope decision with real fan-out (coverage, format, timing) | A micro-choice with no felt difference — route to `docs/decisions.md` |
| Leaf | A concrete mechanism implied by the branch answer | Still open-ended |

Check `docs/decisions.md` before you open a root or trunk in this mode. If it is answered there, cite it.

### technical — an infrastructure or architecture call

Use for a technical or infra subject.

| Level | Pass | Fail |
|---|---|---|
| Root | A binding constraint or quality attribute | Phrased as a customer outcome — use concept-shape instead |
| Trunk | A tradeoff axis where the constraint bites | No real tension — skip to leaf |
| Branch | A competing concrete approach | Two names for the same approach |
| Leaf | The specific implementation choice | Still a category of approaches |

Research before you recommend in this mode.

## Step 1 — Root

Ask the root questions through AskUserQuestion, in batches of 4 to 5. Roots set the shape of the rest of the tree. Continue until every root passes the semantic gate for "root" in the active mode.

A small task can resolve at root alone.

## Steps 2+ — Trunk, Branch, Leaf

For the current open thread, repeat:

1. Read the parent's answer from `decision-tree.md`. Restate it in the next question.
2. List candidate nodes for this level, based on the parent's answer.
3. Run the structural gate and the mode's semantic gate on each candidate.
4. Ask the nodes that pass, through AskUserQuestion.
5. Add the resolved nodes to `decision-tree.md` before you open the next thread.

## Late-Discovered Decisions

A decision found mid-session is not a separate channel. Add it as a new node under its real parent in `decision-tree.md`. Ask it through the same steps: read the parent, gate it, ask it. Do not ask a bare question outside the tree.

## Question Craft

- Ask through AskUserQuestion, in batches of 4 to 5. Use plain text only for orientation or one open-ended push. Do not use a numbered "which one?" list.
- State your recommended answer on every question, with the reason.
- Take a position. Do not say "that could work." Say whether it works, and what would change your mind.
- Do not ask what a file already answers. Read the codebase and prior docs first.
- Research before you recommend anything technical or factual.
- Cite benchmarks and studies as reference points, not as a verdict.

## Stop Condition

Stop only when the tree is complete: every open thread is a resolved leaf or a correctly collapsed node, with no open ambiguity. Before you stop, check for an open thread, or an answer that raised a question you did not ask.
