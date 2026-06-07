# claude-code-sdd

A Claude Code skill package that helps non-developers build bigger projects by enforcing SDD (Spec-Driven Development) practices — for both you and Claude.

Big vibe-code projects feel more like a real structured collaboration instead of a long-running conversation with weird tangents. Every feature follows a clear spec → build → test → dogfood flow, and Claude can't skip any of it.

---

## What it does

**Spec-Driven Development (SDD)** means agreeing on what you're building before building it, then verifying it against that agreement before shipping. Four phases, each behind a gate Claude cannot skip.

**1. Specify**
You approve a ten-line **Outcome Card** — what you'll be able to do at the end of the phase, and what "it worked" looks like on screen. Claude then writes the technical specs against it with parallel drafter agents, and a panel of three adversarial skeptics attacks them (completeness, testability, scope creep) until the holes are closed. You never read a spec file — the card is your contract, and it's what gets graded at the end.

**2. Design**
Your Pencil or Figma file is the source of truth. Claude reads it frame by frame and builds from it directly — from your actual design, not from what looks similar in the existing codebase.

**3. Build + Visual Compliance**
For every UI piece, before Claude saves the work, it screenshots what it made, puts it next to your design frame, and fixes what drifted. Tests are hardcoded scripts via [tdd-guard](https://github.com/nizos/tdd-guard) so Claude can't just skip them when building.

**4. Dogfood Review**
A separate skill uses the `browse` tool to click through every single user story — plus a fleet of three *blind* reviewers (a first-timer, an impatient phone user, a returning user) who only know your goal, never the implementation. An issue flagged by two of three is real; the report leads with a verdict on each Outcome Card promise.

### Built-in wiki

Each skill keeps its own memory at `~/.claude/wiki/`. Claude learns from all your projects and doesn't fall into the same gotchas twice. Built in, no setup, just markdown files on your machine.

---

## Who this is for

Non-developers who want to vibe-code larger projects. Founders, PMs, designers building real products with Claude Code but not writing the code themselves.

The bigger the project, the more Claude drifts without structure. If you're past the "quick prototype" stage and things keep breaking in ways you can't debug, this is for you.

Teams with an existing dev process don't need this. Neither do developers who want full manual control, or people building throwaway scripts.

---

## Known limitations

**Don't try this on Claude Pro.** This workflow is about 18-20% more token-heavy than a normal Claude Code build. You'll hit rate limits mid-build and the flow will stall. Max plan minimum.

**This is v0.5.0.** Gates will break. Flows will get stuck. When they do, file an issue and I'll fix it fast.

---

## Recommended design tool: [pencil.dev](https://pencil.dev)

Free Claude Code wrapper for design. If you don't have a design tool yet, start here. The design gate works best when Claude reads a real Pencil file via MCP — direct frame access is more reliable than exporting images.

Figma works too. The gate doesn't care which tool you use.

---

## Install

**Step 1 — Install Claude Code**
https://claude.com/claude-code

**Step 2 — Install this plugin**
```
/plugin install github:anhtrinh919/claude-code-sdd
```

**Step 3 — Optional but worth it**

TDD guard for logic-heavy phases:
```
/plugin install github:nizos/tdd-guard
```

`/build` checks for missing pieces on first run and gives you the install command for anything that's not there. Nothing is blocking — the flow degrades gracefully.

---

## Starting your first project

In any folder (even empty):

```
cd ~/my-new-app
claude
/build
```

`/build` detects it's a new project and hands off to `/ba`, which drills you on what you're building and who it's for. Ten minutes later you have a constitution: mission, tech stack, roadmap. From there it's one phase at a time.

Already have a `mission.md`? `/build` reads your state file and picks up from the last approved gate.

---

## How the workflow runs

```
/build
  → /ba        scope drill, user stories, screen inventory
  → you approve the Outcome Card (your contract for the phase)
  → /spec      parallel drafters write requirements + plan + validation
               → 3 adversarial skeptics attack the docs → auto-proceeds
  → /frontend  design brief → you design
               (meanwhile: the non-visual backend builds itself in the background)
  → you approve the design
  → /backend   builds from design in dependency waves, visual compliance on every UI commit
  → /review    spec check + blind 3-persona reviewer fleet + Outcome Card grading
  → you test the exact promises you approved on the card
```

You only ever approve outcomes — the idea, the card, the design, the working app. Everything technical is validated by adversarial machine gates instead of asking you to read specs. And if your session crashes mid-phase, run `/build` again — it picks up at the last gate, no starting over.

---

## The seven skills

| Skill | What it does |
|---|---|
| `/build` | Entry point. Reads state, routes to the right next skill, writes `.build-state.json` at every gate. |
| `/ba` | Drills you on scope until user stories and screen inventory are real, not vague. Ends with the Outcome Card — the only thing you approve at spec time. Competitor research runs in the background while you answer. |
| `/spec` | Parallel drafters write `requirements.md`, `plan.md`, `validation.md`; a 3-skeptic panel validates them against your card, then the pipeline auto-proceeds. |
| `/frontend` | Design brief, handover to your design tool, extracts tokens and frame references into `handover.md`. |
| `/backend` | Reads the design file directly. Implements in small task groups. Visual compliance before each commit. |
| `/code-harness` | Called by `/backend` on every task group. Gates each change behind a spec contract and a verify script. |
| `/review` | Validation checklist, then a blind 3-persona reviewer fleet walks the app (2-of-3 rule). Report leads with a per-outcome verdict on your card. |
| `/bad-idea` | Pressure-tests your idea before any building starts. Seven lenses, adversarial by default. Also auto-runs at the start of every new project. |
| `/adversarial-review` | Structural critique of what was just built. Attacks module shape, abstraction necessity, and naming. Called by `/backend`; also user-invocable. |
| `/dogfood` | Browser-driven testing of anything you just built outside the `/build` pipeline. Three-signal gate — doesn't report "done" until the user can actually accomplish their goal. |

---

## FAQ

**Do I need to know how to code?**
No. You need to know what you want and be able to say "yes, that's it" or "no, that's wrong." Claude handles every technical decision that doesn't change what your users experience.

**What happens if I just tell Claude to start coding without `/build`?**
Claude will build something. It just won't be what you meant. That's the entire problem this plugin solves.

**Can I use Figma instead of Pencil?**
Yes. The gate is tool-agnostic. Pencil has the tightest MCP integration right now, but any design source Claude can read will work.

**What actually happens at a gate?**
Claude writes down what it's about to do and asks you to approve. Say no and it revises. Say yes and the state file updates — so if the session crashes, it can recover from exactly that point.

**How do I recover if `/build` gets confused?**
Check `.build-state.json` in the project root. It shows the last approved gate. Delete it to restart the current phase, or edit the `step` field to jump back.

---

## Troubleshooting

**`/build` keeps re-asking things you already answered.**
`.build-state.json` was reset or deleted. Check that it exists and has the `step` you expect.

**Wiki errors in the logs.**
Informational only. Nothing's blocked.

**`/review` says "no validation spec found."**
You're running it outside a phase directory. Run `/spec` for this phase first.

**Hook conflicts with another plugin.**
Claude Code fires all plugin hooks. If two plugins register the same trigger, both fire. Disable the conflicting one in settings.

---

## Bundled tools

- `scripts/wiki.mjs` — per-skill memory store. Skills auto-call it; you don't have to.
- `bin/tdd-config` — the **only** sanctioned way to toggle tdd-guard config per phase. Hand-editing `.claude/tdd-guard/data/config.json` wipes the project's accumulated `ignorePatterns`. The wrapper merges instead. See `docs/INSTALL-DEPS.md` for usage.

---

## Contributing

MIT. Issues and PRs: https://github.com/anhtrinh919/claude-code-sdd
