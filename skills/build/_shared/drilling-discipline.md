# Drilling discipline

Applies to any drilling session in this stack — concept interview, product interview, or phase scope grill. Load-bearing; keep verbatim when copied or pointed to.

This is a **relentless interview**, not a checklist. A question count hit is never the reason to stop; an unresolved branch is always the reason to continue.

## The four-step shape

Non-trivial drilling runs in four steps, in order. Each produces an artifact the user can correct before the next begins — that is what stops drift, and it is why this beats question-by-question interrogation.

**Step 1 — Batched orientation.** Establish context and the genuinely big decisions through **`AskUserQuestion` in batches of 4–5 related questions per call**. These are the decisions that determine what the tree even looks like — get them wrong and every question after them is misplaced. Keep going in batches until you can name the roots. This step is conversational and fast; do not attempt completeness here.

**Step 2 — Author `decision-tree.md`.** Derive the real structure from what Step 1 established and write it to disk. Every remaining open decision goes in it, grouped, with the dependency edges named. Run the validity tests below **before** showing it. Present it and let the user correct the *structure* — this is the cheapest possible moment to discover the tree is wrong.

**Step 3 — Convert the tree into a fillable artifact.** Turn every open decision into a form the user completes in one sitting and hands back. Requirements in "The artifact" below.

**Step 4 — Analyze the returned answers.** Read them as a set, not one at a time: look for answers that contradict each other, options the user rejected via *Other* (the option list was wrong — say so), notes that reveal a question you failed to ask, and deliberate blanks (a blank is data). Surface conflicts before folding anything into docs.

**Sizing.** A five-minute tweak resolves in Step 1 alone — do not manufacture a tree for it. Steps 2–4 earn their cost once the open decisions exceed roughly a dozen, or whenever the user cannot see the whole shape at once. When in doubt, run all four: the failure mode this replaces is a long question-by-question session in which the user answers thirty things and still cannot picture the product.

## Deriving the tree — the validity tests

**A tree is derived, never imposed.** Picking depth levels and sorting questions into them produces a rectangle, not a hierarchy. Run all four tests before showing any tree; failing any one means re-derive, not adjust.

- **Parent test.** X is a parent of Y only if changing X's answer changes Y's answer, its options, or whether Y exists at all. If Y survives any answer to X unchanged, they are siblings. This is the only definition of a parent.
- **Fan-out test.** Each level must be materially wider than the one above. Roughly equal levels are the signature of tiers invented first and populated afterwards.
- **Single-child test.** A node with exactly one child is not a branch point — it is the same decision restated. Collapse it into its parent. A chain of single-child nodes is a linear sequence drawn vertically, and it conveys nothing.
- **Settled-fact test.** A decision already made that nothing else depends on is not a root. Pull it out of the tree into a flat "settled" list. Leaving it in inflates the top of the tree and hides the real roots.

**When genuine dependencies turn out to be rare — which is the common case — say so and use the honest structure:** a few **roots** (the load-bearing bets or constraints), each holding a small number of **decision areas**, each holding several questions that are genuinely parallel. Hierarchy is real from root to area; inside an area everything is siblings, answerable in any order. Record the few real cross-area dependencies as explicit **edges** in prose. Never fake nesting to make a diagram look like a tree.

**Inversion check.** Never leave a child decided while its parent is open. If it happens, mark it in the doc as answered-before-its-parent rather than quietly hiding it.

**If the user says the questions feel random, fringe, scattered, or disconnected — stop immediately.** That is the signature of a wrong tree, not of bad individual questions. Return to the roots and re-derive; do not patch the current question or apologise and continue.

## `decision-tree.md`

Written to the project root (or alongside the phase docs). Contains:

- The **roots**, each one sentence, with why it is a root — what breaks if it is answered differently.
- The **areas** under each root, each with a one-line scope note.
- Every **open decision** in each area, as a question.
- **Edges** — the cross-area dependencies, in prose.
- **Settled, with nothing depending on it** — a flat table, so it is not re-litigated.
- **Sequencing** — which areas block which. An empty area hanging off a root is the most urgent thing in the document; call it out explicitly.

Also record, briefly, any tree you rejected and why. It stops the next session re-deriving the same wrong shape.

## The artifact

Built with the `artifact-design` skill and published via the `Artifact` tool. It must:

- Group questions by root → area, preserving the tree's structure so the user sees where each question sits.
- Give every question **real options with a one-line consequence each**, the recommended one flagged and placed first. Describe what each option *costs*, not why it is appealing.
- Provide an **Other** free-text field on every question, and a **note** field for when the option list itself is wrong. Say plainly that the note field is the most valuable one.
- Show **progress**, and let the user **skip** freely — record blanks and return them in the export.
- **Autosave** to `localStorage` so the session survives a closed tab.
- **Export** to plain text the user can paste back, including the list of what was left blank.

Never inline a numbered question list in chat for the user to type answers to.

## Question craft

- **Always ask via `AskUserQuestion`** in Step 1, batched 4–5 at a time. Plain text is only for orientation or a single open-ended push. Inline numbered "which one?" lists are forbidden everywhere.
- **Every question carries your recommended answer** — not just felt-impact forks. State your default and why; the user reacts to it instead of drafting from a blank page. This is what makes a deep session tractable instead of exhausting.
- **Take a position.** "That could work" is banned — say whether it will work and what evidence would change your mind.
- **Never ask what a file already answers.** Read the codebase, `docs/decisions.md`, and prior docs first.
- **Research before recommending** on anything technical or factual. A recommendation with no grounding is noise the user has to fact-check.
- **Benchmarks and studies are reference points, not verdicts.** Cite them to inform a fork, never to declare a decision made or an idea dead.

## Stop condition

Tree-completeness, not a count or a duration. Every branch relevant to the topic resolved, no latent ambiguity. If tempted to wrap up, ask: is there a branch I opened and didn't close, or an answer that implied a question I never asked?
