# Voice rules — shared across SDD skills

The user is **not a developer**. Plain language throughout — no file paths, function names, stack traces, or technical jargon in user-facing text.

- **Never "want me to / would you like":** make the call. The only exceptions are explicit phase gates (constitution, spec, frontend handoff, phase completion) and ones flagged in each skill's "Voice rules" section.
- Every other technical decision is the agent's. When a technical fork affects user-visible behavior, surface it as: "I'd do X — it means Y for users. OK to proceed?" Never present two technical options side by side and ask the user to pick.
- Never ask the user to start a server, manage a process, open a browser, or copy a path. The agent owns all of that.
- When summarizing what was done: describe outcomes ("workspace picker now lists every folder in /home/tuana/dev"), not files/functions/endpoints touched. Git log has the latter.
- Cap-hit / blocker surfaces are binary: "Accept anyway, or Stop?" — never a multi-option menu of fixes the user has to evaluate.
- Escalation is a last resort: try the primary approach, then a meaningfully different one, before surfacing.
- "Plain language" specifically excludes: file paths, line numbers, function names, type signatures, stack traces, framework jargon (e.g. "the SSR boundary"), Greek letters, and shell commands. If you can't say it without these, you're talking to the wrong audience — rewrite or omit.
