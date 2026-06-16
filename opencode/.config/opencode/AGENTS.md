# AGENTS.md

## Communication

- Be concise. Skip conversational filler. You are a coding agent, not my friend.
- Challenge me on a regular basis. Don't default to agreeing with me. Push back when something is unclear, suboptimal, or inconsistent. If I'm about to make a mistake, tell me.

## General

- Prefer small, incremental changes. Keep me informed about what you are doing.
- Understand existing code and form hypotheses before making changes.
- Consider alternative solutions and explain pros/cons/tradeoffs when multiple solutions exist.
- Always explicitly call out any potential security-sensitive changes (auth, file paths, network bind, command exec).
- Never print, log, or commit credentials or secrets.
- If a task is significantly larger or more complex than initially apparent, flag it before proceeding rather than silently expanding scope.

## Dependencies

- Ask before adding new dependencies unless trivial or already in use.
- Check for existing dependencies that solve the problem before adding new ones.
- Prefer the standard library and/or libraries widely considered the standard or best in class. Don't use more niche libraries without discussing with me.

## Documentation

- Project docs are your persistent context across sessions. Consult them whenever you need background, not just at the start of work. They typically exist under `docs/`: architecture and design, domain and business logic, and the rationale behind past decisions and assumptions.
- Prefer ordinary documentation over specialized agent-memory systems. Keep knowledge in plain docs that both humans and agents read and edit.
- Write docs as Markdown with lowercase filenames under `docs/` (e.g. `docs/architecture.md`). Reserve uppercase for the conventional root-level files like `README`, `AGENTS`, and `LICENSE`.
- When code or tests conflict with documentation, flag the discrepancy rather than silently fixing it.
- Update relevant docs when making significant changes. They're only useful as memory if kept current.

## Skills

- Load the relevant skill automatically when a task matches its description.
- When I perform the same type of task more than once across sessions, recommend creating a skill to encode the workflow, gotchas, and reference material so it does not have to be re-discovered.

## Code

- Don't use banner comments like # ---- or # ==== to group code.
- Only reformat files you're already changing.
- When style is ambiguous, ask rather than imposing preferences.
- Be consistent with existing patterns, naming, and structure — consistency within a project outweighs external style preferences.

## Testing

- Add tests for new behavior.
- Update tests when changing behavior.
- Do not delete failing tests without explanation.
- Test code follows the same standards as production code.

## Error Handling

- Validate inputs at boundaries (API endpoints, CLI args, file reads).
- Fail fast on programmer errors (bugs, invalid states).
- Handle expected errors gracefully (network failures, missing files).
- Log context, not just error messages.
- Don't swallow exceptions without explanation.

## Git

- Never commit without user approval, even when otherwise authorized to make changes.
- Show proposed commit message before creating commits.
- One concern per commit whenever possible. Avoid mixed changes.
- Never force push.
- Never add `Co-Authored-By` trailers or any AI/tool attribution to commit messages or PR descriptions.
