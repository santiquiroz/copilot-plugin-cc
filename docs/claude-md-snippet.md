# CLAUDE.md snippet

Paste the block below into your `~/.claude/CLAUDE.md` (or a project
`CLAUDE.md`) to make Claude Code delegate mechanical work to this plugin
proactively. Adjust the trigger table to your stack. If you also run a
reasoning-capable delegate (e.g. Codex), see
[docs/delegation-guide.md](delegation-guide.md) for the full split — this
snippet only covers the Copilot lane.

```markdown
# Copilot CLI delegation (copilot plugin)

Mandatory, no user prompt needed: before writing code, decide explicitly —
Copilot (mechanical), a reasoning delegate (e.g. Codex), or inline. Inline
only for the never-delegate list below or trivial edits (<5 lines, 1 file)
where coordinating delegation costs more than doing it. Most qualifying
subtasks should go to a delegate, not inline — a task that ends up 100%
inline despite qualifying for delegation is a process miss, not a shortcut.

Copilot owns pure-mechanical, zero-domain-context work. Delegate proactively:

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
- Quota fallback: a 429/rate-limit error carrying a retry-after hint is a
  soft cap — wait that long and retry the SAME delegate once. A 5xx/overload
  error with no retry hint → fail over immediately (Copilot quota out → try
  the reasoning delegate, and vice versa) rather than waiting.
- Copilot-specific bug: resuming a session after a rate-limit hit can hang.
  On "rate limit" in Copilot output + ~10s of silence: kill the process,
  don't wait, relaunch fresh without `--continue`.
- If both lanes are quota-exhausted in the same session: stop auto-delegating
  for the rest of the session, handle everything inline, and mention this
  once.
```
