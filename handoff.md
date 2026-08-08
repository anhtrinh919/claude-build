# Handoff — pick up work on the claude-build plugin

> Read fully. This restates the skill-shortening rules and this session's gotchas verbatim
> from auto-memory, because that memory is scoped per-project
> (`~/.claude/projects/-home-tuananh--claude/memory/`) and won't carry over into this repo's
> session.

## Brief

User: "write /handoff for new session to work with build stack (intentionally exclude from
this session)." The prior session's work was entirely inside `~/.claude` (dook skill
overhaul, `browse` SKILL.md rewrite, `dook-review` shortening — committed and pushed there).
This repo (`~/dev/claude-build`, the `/build` and `/build-lite` orchestrator plugin) was
explicitly kept out of that session.

## Skill-shortening rules (verbatim from auto-memory)

These four notes govern any pass that rewrites or condenses a Claude Code skill file
(`SKILL.md`, `references/*.md`). Quoted as-written from
`~/.claude/projects/-home-tuananh--claude/memory/`.

### asd-ste100-skill-rewrite

> For each skill file the user names, one at a time this session, rewrite it in ASD-STE100
> (Simplified Technical English: short sentences, active voice, imperative mood, one
> word/one meaning, no stacked noun clusters, no gerunds-as-nouns) under two separate
> character budgets:
>
> - **Frontmatter `description`**: max 300 chars, every time. Keep literal trigger phrases
>   (e.g. `/grill-me`, `"grill me on this"`) verbatim — skill-matching needs the exact
>   string, not STE prose.
> - **Body**: to whatever budget the user gives for that specific file (has varied: 2000,
>   10000, etc.). Verify with `wc -c` and iterate down until it fits — do not eyeball it.
>
> Cut aggressively, but never lose fidelity: drop micromanaging instructions,
> self-justifying "mansplaining" prose (explaining *why* a rule exists when the rule alone
> suffices), retired/historical context that no longer applies (e.g. a decommissioned
> machine's paths), and any reference left dangling by an earlier cut. Keep every actual
> mechanism, gate, command, path, and conflict rule. Code blocks and exact commands/paths
> are never rewritten to STE — only prose is.
>
> Section ordering rule (see below): concept-defining sections (an artifact, a gate, a data
> structure) go before the procedure sections that reference them — never interleave
> definition and procedure.

### skill-section-ordering

> When rewriting or condensing a skill's markdown (SKILL.md or a references/*.md file),
> order sections so anything that names/defines an artifact or gate (e.g.
> "decision-tree.md", "the structural gate") appears before the procedural steps that
> reference it. Don't interleave concept-definition sections with procedure sections.

### no-nibbling-char-budgets

> When a file is over a character budget, estimate how much must come out and cut that
> much in one or two bold edits — don't nibble: small trim, `wc -c`, small trim, `wc -c`,
> repeated many times.
>
> **How to apply:** compute the gap (current size − budget) up front. If the gap is large
> (a few hundred+ chars), cut whole sentences, clauses, or examples in one edit sized to
> close most of the gap, then verify with `wc -c` once. Reserve small follow-up trims only
> for closing the last small remainder, not as the default method.

### no-hard-wrap-markdown

> When writing or editing a markdown file, never hard-wrap a paragraph or bullet across
> multiple source lines at a fixed column width. Write each paragraph, bullet item, or step
> block as a single line, however long.
>
> **How to apply:** when authoring or rewriting any `.md` file (skills, docs, references),
> keep code blocks/tables as-is but write surrounding prose paragraphs and list items
> unwrapped — one line per logical unit. Never use a text-wrap script or column-width fill
> on markdown prose.

## This session's gotchas (specific, non-obvious, not yet in a memory note)

1. **Frontmatter `description` has its own char budget, separate from the body** — ≤300 chars,
   verified with `wc -c`, every time a skill file is touched. Easy to forget mid-rewrite: this
   session missed it on `dook-review/SKILL.md` (left at ~980 chars) and `dook/SKILL.md` (689
   chars) — both slipped through a full body rewrite and were only caught when the user asked
   "did you forget my rules? recheck memory." Check both, every time.

2. **Char-budget cuts hit a real fidelity floor on dense technical docs.** Rewriting a
   schema/business-rules reference (`bq-knowledge.md`) could only reach ~21% cut without
   trading away load-bearing content (calibration numbers, specific examples, thresholds) —
   nearly every sentence carried a distinct fact. A target range (e.g. "30-70% cut") isn't
   always reachable "without losing fidelity." When it isn't: stop, report the actual number,
   and ask whether to accept it or explicitly trade fidelity for size — don't silently
   under-deliver.

3. **"Remove PROCESS/OPINION" rewrites are a different move from char-budget cuts.** When the
   ask is to strip process/opinion rather than shrink under a budget, distinguish real
   technique/mechanism/security rules (keep) from prescribed workflows and recommendations
   (cut) — e.g. a numbered "how to test a feature" walkthrough is process and goes, but a
   prompt-injection defense rule or a plan-mode permission list is a rule, not opinion, and
   stays even in that kind of pass.

4. **`~/.claude` is a large repo shared across machines** — `git status` there can show
   hundreds of files that look unrelated to the current task (brain/memory sync, prior-session
   deletions, directory renames). That's legitimate, not a reason to second-guess an explicit
   "commit & push," but still worth a quick scan for anything secret-shaped before pushing.

## Decisions

None — out of scope for the prior session.

## Corrections — do not repeat these

- Don't interpret "do the cut pass" as dedup-only when the user's actual ask is a char-count
  reduction — this session did exactly that once, and the user had to correct it ("i meant,
  can we cut more chars... same deal as before"). Confirm which is meant, or ask.
- Don't forget the frontmatter `description` budget when rewriting a skill body — see gotcha 1.

## Still open

No specific build-stack task is defined yet. Don't guess one from the CHANGELOG history — ask
the user what to work on before making changes.

## Next step

Read `README.md` and the last 2-3 `CHANGELOG.md` entries, then ask the user what they want
done here. If the work touches any `skills/build*/SKILL.md` or `references/*.md` file, the
char-budget/section-ordering/hard-wrap gotchas above apply directly.
