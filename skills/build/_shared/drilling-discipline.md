<!-- SYNCED FROM ~/.claude/skills/grill-me/references/drilling-discipline.md — do not hand-edit. Re-run sync-drilling-discipline.sh after any change to the canonical file. Source hash: 99a1ecbedec1 -->

# Drilling discipline

Applies to any drilling session across grill-me, build-spec, and build-lite-spec. **Canonical copy.** The two plugin repos carry a generated copy of this exact file — never hand-edit theirs; run `sync-drilling-discipline.sh` after any change here.

This is a **relentless interview**, not a checklist. A question count hit is never the reason to stop; an unresolved node is always the reason to continue.

## The four levels, walked one thread at a time

Every open decision sits at one of four levels: **root → trunk → branch → leaf**. A level exists only where a real dependency lives there — it is never manufactured to complete the shape. Most sessions don't reach leaf on every thread; many don't reach branch. That's correct, not incomplete.

- **Depth-first, one open thread at a time.** Settle a branch's leaves fully before its sibling branch. Settle a trunk's branches fully before its sibling trunk. Settle a root's trunks fully before the next root. Never open a second thread while the first is unresolved — this is what makes the traversal itself an anti-drift device, not just a nice-to-have.
- **Root is the one batched exception.** Roots are the few, mostly-independent load-bearing bets — by definition, a decision that depends on another root isn't a root, it's that root's child. Resolve all roots together in one batched orientation (Step 1 below), not one at a time.
- **A node is entered at a level only if it passes both gates:**
  - **Structural gate** (same for every mode) — the four validity tests below.
  - **Semantic gate** (mode-specific — see "Gate presets").
  Failing either: collapse straight to the next real level down, ask it as a single plain question with no tree ceremony, or — if nothing depends on it — pull it into the flat settled list. Never invent a level to keep root/trunk/branch/leaf symmetrical; an unvalidated tree-shaped ritual is exactly the failure this replaces.
- **A level is derived from, and only from, the answer just given to its parent.** A trunk's branches cannot be authored before the trunk is answered; a branch's leaves cannot be authored before the branch is answered. This is not a simplification — it's what "parent" means. Authoring a child before its parent is answered is authoring a guess, and it is the direct cause of drift (no committed structure to return to) and of dead artifact rows (children guessed for a parent answer that didn't happen).

## Step 1 — Root: batched orientation

Establish the roots via **`AskUserQuestion` in batches of 4–5 related questions per call**. These are the decisions that determine what the rest of the tree even looks like — get them wrong and everything after is misplaced. Keep batching until every root passes the semantic gate for "root" in the active mode (see "Gate presets"). Fast and conversational; do not attempt trunk/branch/leaf completeness here.

**Sizing.** A five-minute tweak resolves in root alone — never manufacture depth for it. Deeper levels earn their cost once a root's open decisions genuinely exceed roughly a dozen, or whenever the user can't see the whole shape at once.

## Steps 2+ — Trunk, branch, leaf: reveal, gate, ask

Repeat this for each level, for the current open thread only:

1. **Read the parent answer back from `decision-tree.md`** — never from conversational memory — and open the prompt by restating it ("Given you chose X for the root, the open question at trunk is..."). This is the drift fix made mechanical: there is no path to asking a question that assumes something the file doesn't record, because the prompt is generated from the file.
2. **Enumerate candidate nodes at this level**, conditioned on the parent answer. Run the structural gate (validity tests) and the active mode's semantic gate on each candidate.
3. **Nodes that survive both gates** get asked — via a small `AskUserQuestion` batch, or the fillable-artifact treatment once the set is large enough to earn the format (same sizing rule as Step 1).
4. **Append the resolved nodes to `decision-tree.md`** before moving to the next thread. The file is written to incrementally over the session, never authored once upfront.

## Deriving the tree — the validity tests (structural gate)

**A tree is derived, never imposed.** Picking depth levels and sorting questions into them produces a rectangle, not a hierarchy. Run all four tests before entering any level; failing any one means collapse the level, not adjust the question.

- **Parent test.** X is a parent of Y only if changing X's answer changes Y's answer, its options, or whether Y exists at all. If Y survives any answer to X unchanged, they are siblings, not parent/child.
- **Fan-out test.** Each level must be materially wider than the one above. Roughly equal levels are the signature of tiers invented first and populated afterwards.
- **Single-child test.** A node with exactly one child is not a branch point — it is the same decision restated. Collapse it into its parent; don't draw the level. A chain of single-child nodes is a linear sequence drawn vertically, and it conveys nothing.
- **Settled-fact test.** A decision already made that nothing else depends on is not a root (or trunk, or branch). Pull it out of the tree into a flat "settled" list. Leaving it in inflates the tree and hides the real structure.

**When genuine dependencies turn out to be rare — the common case — say so and use the honest structure:** a few roots, each holding only the trunks that genuinely fork from it, each holding only the branches that genuinely fork from that. Record the few real cross-thread dependencies as explicit **edges** in prose. Never fake nesting to make a diagram look like a tree.

**Inversion check.** Never leave a child decided while its parent is open. If it happens, mark it in the doc as answered-before-its-parent rather than quietly hiding it.

**If the user says the questions feel random, fringe, scattered, or disconnected — stop immediately.** That is the signature of a wrong tree, not of bad individual questions. Return to the roots and re-derive; do not patch the current question or apologise and continue.

## Gate presets — what counts as each level, by mode

The structural gate above is mode-neutral. What a node has to *be* to count as a root/trunk/branch/leaf is not — it depends on what kind of thing is being decided. A call site declares which preset applies; it is never auto-detected mid-session. (grill-me is the one general-purpose, multi-subject call site — it classifies its own mode at the top of its flow before Step 1 fires; see grill-me/SKILL.md.)

### `concept-shape` — a new idea, nothing decided yet
Use for: a brand-new project/feature's concept interview (build-shape / build-lite-shape Step 1.1; grill-me when the subject is a new idea).

| Level | Passes the gate if... | Fails if... |
|---|---|---|
| Root | It's the outcome or existential bet — the "why," answerable independent of anything else | It's itself explained by something above it, or it's already a deliverable ("build X") |
| Trunk | It names a specific actor/stakeholder, or a specific unmet need standing between now and the root | The answer is already a feature name — that's a branch or leaf wearing trunk clothing |
| Branch | It's a genuine behavior-level fork — ≥2 live options that change what gets built | There's really only one live option (single-child test) |
| Leaf | It's concrete and buildable, no "how would we do that" left open | An open "how" remains — drill one more level, or it isn't a leaf yet |

### `phase-scope` — scoping a slice of an already-decided product
Use for: build-spec / build-lite-spec phase mode, and Step 1.4's product interview once demand/actor are already settled by shape.

| Level | Passes the gate if... | Fails if... |
|---|---|---|
| Root | It's the phase's Outcome Card outcome | N/A — always cited from the outcome card, never re-derived here |
| Trunk | — | **Always fails by design.** The actor/need was already settled upstream; re-deriving it here re-litigates a settled decision. Collapse straight to branch. |
| Branch | It's an open scope decision with real fan-out — coverage, format, timing (e.g. all-cards-vs-per-card, sync-vs-async) | It's actually a technical micro-choice with no felt difference — route it to `docs/decisions.md` instead, don't ask it |
| Leaf | It's a concrete mechanism or parameter choice implied by the branch answer | Still open-ended — not a leaf yet |

Before opening any root or trunk in this mode, check `docs/decisions.md` first — if it's already answered, cite it, never re-ask.

### `technical` — an infrastructure or architecture call
Use for: grill-me on a technical/infra subject; any drilling of a pure implementation choice inside `/build`(-lite) that isn't covered by the phase's own scope grill.

| Level | Passes the gate if... | Fails if... |
|---|---|---|
| Root | It's a binding constraint or quality attribute ("must not block the UI past 100ms", "must run offline") | It's phrased as a customer outcome or an actor's need — that's `concept-shape`, not this mode |
| Trunk | It's a tradeoff axis where the constraint actually bites ("cost lives client-side vs. server-side") | There's no real tension — the constraint is satisfied one obvious way; skip to leaf |
| Branch | It's a competing concrete approach (a named algorithm, architecture, or pattern) | Two "approaches" are actually the same thing under different names |
| Leaf | It's the specific implementation choice / config | Still a category of approaches, not a single choice |

Research before recommending in this mode more than any other — an ungrounded technical recommendation is noise the user has to fact-check, and this mode exists precisely because product-shape question craft (react-to-my-default) doesn't fit architecture tradeoffs the same way.

## Latent / late-discovered decisions

Anything discovered mid-session — a scope inference, a technical micro-choice, a felt-impact fork surfaced while authoring docs — is **not a separate channel**. Append it as a new node under its real parent in `decision-tree.md` (usually a leaf, since it's being discovered this late) and ask it through the same mechanism above: read the parent back from the file, gate it, ask it. Never fire a bare `AskUserQuestion` outside the tree — that reopens the exact "asks a question that's out of the tree" failure this discipline exists to close.

## `decision-tree.md`

A **living document**, appended to as each level resolves — not authored once upfront. Written to the project root (or alongside the phase docs). Contains:

- The **roots**, each one sentence, with why it is a root — what breaks if it is answered differently.
- Under each root, its **trunks** (if any survived the gates), each with a one-line scope note.
- Under each trunk, its **branches** (if any), each with a one-line scope note.
- Every open **leaf** decision, as a question.
- **Edges** — cross-thread dependencies, in prose.
- **Settled, with nothing depending on it** — a flat table, so it is not re-litigated.
- **Sequencing** — which threads block which. An empty thread hanging off a resolved parent is the most urgent thing in the document; call it out explicitly.

Also record, briefly, any node you collapsed or rejected and why — it stops the next session re-deriving the same wrong shape.

## The artifact

Built with the `artifact-design` skill and published via the `Artifact` tool, once a level's node set earns the format (see sizing rule). It must:

- Group questions by the thread they sit under, preserving root→trunk→branch context so the user sees where each question sits.
- Give every question **real options with a one-line consequence each**, the recommended one flagged and placed first. Describe what each option *costs*, not why it's appealing.
- Provide an **Other** free-text field on every question, and a **note** field for when the option list itself is wrong — say plainly that the note field is the most valuable one.
- Show **progress**, and let the user **skip** freely — record blanks and return them in the export.
- **Autosave** to `localStorage` so the session survives a closed tab.
- **Export** to plain text the user can paste back, including the list of what was left blank.
- **Redeploy the same URL as new levels unlock**, appending the newly-revealed layer, rather than publishing a fresh link per level — one evolving link across the whole session.

Never inline a numbered question list in chat for the user to type answers to.

## Question craft

- **Always ask via `AskUserQuestion`**, batched 4–5 at a time within a level. Plain text is only for orientation or a single open-ended push. Inline numbered "which one?" lists are forbidden everywhere.
- **Every question carries your recommended answer** — state your default and why; the user reacts to it instead of drafting from a blank page.
- **Take a position.** "That could work" is banned — say whether it will work and what evidence would change your mind.
- **Never ask what a file already answers.** Read the codebase, `docs/decisions.md`, and prior docs first.
- **Research before recommending** on anything technical or factual. A recommendation with no grounding is noise the user has to fact-check.
- **Benchmarks and studies are reference points, not verdicts.** Cite them to inform a fork, never to declare a decision made or an idea dead.

## Stop condition

Tree-completeness, not a count or a duration: every open thread either resolved to a leaf or correctly collapsed by a gate, no latent ambiguity. If tempted to wrap up, ask: is there a thread I opened and didn't close, or an answer that implied a question I never asked?
