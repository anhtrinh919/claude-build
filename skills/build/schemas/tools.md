# tools.md — Schema

> Agent context — not for human reading.
>
> Project-root file. The agent's quick-reference for "where does X live, what's the IP, what's the GitHub repo, what Cloudflare zone is this on" — so a fresh session does not re-search Tailscale, re-grep SSH config, or re-ask the user about the GitHub repo URL every time.

## Purpose

Real credentials stay in `.env` files (gitignored). `tools.md` holds the **public-facing pointers and identifiers** an agent needs to operate: machine IPs, account usernames, repo URLs, deployment targets, env var names, service identifiers. Reading this file once per session beats re-deriving it from scratch.

## File location

`<project-root>/tools.md` — alongside `mission.md`, `roadmap.md`, etc.

## Seeding

`/spec` Mode 1 (constitution write) creates this file by:

1. Reading `~/.claude/tools.md` (the user's global master).
2. Copying the `## Machines`, `## Accounts`, and `## Services` sections verbatim into the project's `tools.md`.
3. Appending an empty `## Project-specific` section as a stub for the user (or agents discovering project tools) to fill.

If `~/.claude/tools.md` is missing on a fresh machine, `/spec` Mode 1 writes only the `## Project-specific` section and surfaces a one-line note: "Global tools.md not found — machine/account sections skipped. Create `~/.claude/tools.md` to seed future projects."

## Template

```markdown
> Agent context — not for human reading.

# Project tools, machines, services

Quick-reference for this project's operational context. Synced from `~/.claude/tools.md` at constitution time — refresh manually if global machines/accounts change.

---

## Machines

*(Copied from ~/.claude/tools.md — refresh if global changes.)*

[machine sections from global tools.md]

---

## Accounts

*(Copied from ~/.claude/tools.md — refresh if global changes.)*

[account sections from global tools.md]

---

## Services

*(Copied from ~/.claude/tools.md — refresh if global changes.)*

[service sections from global tools.md]

---

## Project-specific

Things that apply only to **this** project. Append as discovered.

### GitHub repo
*(URL, default branch, any branch-protection rules worth knowing.)*

### Deploy target
*(Where this project ships to: Cloudflare Pages / Workers project name, Vercel project, Fly app name, custom VPS, etc. Include the live URL once one exists.)*

### Cloudflare
*(If used: zone, account ID, Workers/Pages project name, any non-default routing or DNS notes.)*

### External APIs / integrations
*(Third-party services this project calls: Anthropic API, Stripe, Twilio, etc. Just identifiers and base URLs — NOT keys.)*

### Env vars
*(Names and one-line purpose of every env var this project reads. Where the value lives — `.env`, `.env.dogfood`, Cloudflare secrets binding, etc. NOT the values themselves.)*

### Project-specific paths
*(Non-obvious paths inside the project — DB file location for dogfood, log file paths, cache dirs, etc.)*
```

## Rules

- **No real credentials in this file.** Ever. Values live in `.env` (gitignored). This file lists env var **names** and where their values live, not the values.
- The `## Machines`, `## Accounts`, `## Services` sections are a **snapshot copy** from the global master. If a user changes their Tailscale IP, they update `~/.claude/tools.md` once and either re-sync projects manually or accept that older projects' `tools.md` is slightly stale.
- The `## Project-specific` section is the agent's responsibility to keep current. When a deploy target is added or a new service is integrated, the implementing agent updates this section in the same session.
- Read this file at the start of any session that needs to know "where does this project live" — `/build`'s project state prime reads it on every next-feature entry.
