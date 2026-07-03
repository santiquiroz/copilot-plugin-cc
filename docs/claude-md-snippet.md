# CLAUDE.md snippet

Paste the block below into your `~/.claude/CLAUDE.md` (or a project
`CLAUDE.md`) to make Claude Code delegate mechanical work to this plugin
proactively. Adjust the trigger table to your stack.

```markdown
# Copilot CLI delegation (copilot plugin)

Copilot owns pure-mechanical, zero-domain-context work. Delegate proactively —
no user prompt needed:

| Trigger | Action |
|---|---|
| Creating/updating simple test specs (CRUD/mapping, no branching logic) | `copilot:copilot-rescue` subagent, background |
| Mechanical rename across 3+ files | `copilot:copilot-rescue` subagent, background |
| Dead code removal / unused imports / debug-statement cleanup | `copilot:copilot-rescue` subagent, background |
| Generating interfaces from DTOs/entities (pure field mapping) | `copilot:copilot-rescue` subagent, background |
| Component boilerplate shells (imports, constructor, lifecycle) | `copilot:copilot-rescue` subagent, background |
| Applying an identical config block to N similar files | `copilot:copilot-rescue` subagent, background |
| PR description when commits are self-explanatory | `copilot:copilot-rescue` subagent, background |

Never delegate: domain logic, business rules, architecture decisions, complex
refactors requiring full codebase context, any task where the WHY lives in
this conversation.

Rules:
- Launch delegations in the background and keep working — never idle waiting.
- WIP cap: 3–5 concurrent delegations. Kill-switch: 3 failed iterations on
  the same task → stop retrying, bring it inline.
- On "rate limit" in Copilot output: kill and relaunch fresh (no --continue);
  if quota is exhausted, fall back inline and say so in one line.
```
