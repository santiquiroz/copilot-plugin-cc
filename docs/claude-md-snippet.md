# CLAUDE.md snippet

Paste the block below into your `~/.claude/CLAUDE.md` (or a project
`CLAUDE.md`) to make Claude Code delegate mechanical work to this plugin
proactively. Adjust the trigger table to your stack. If you also run a
reasoning-capable delegate (e.g. Codex), see
[docs/delegation-guide.md](delegation-guide.md) for the full split — this
snippet only covers the Copilot lane. No second delegate at all? Skip to the
[no-reasoning-delegate variant](#no-reasoning-delegate-eg-no-codex) below.

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

## No reasoning delegate (e.g. no Codex)?

If Copilot CLI is your only delegate, fold reasoning-shaped work back into
"keep inline" — there's no second lane to catch it when Copilot can't take a
task or hits quota. Use this variant instead:

```markdown
# Copilot CLI delegation (copilot plugin)

Copilot is the only delegate here. Before writing code, decide: Copilot
(mechanical) or inline (everything else — including build-fixing, multi-file
refactors, and anything needing real reasoning, since there's no reasoning
delegate to hand those to). Inline only for the never-delegate list below or
trivial edits (<5 lines, 1 file) where coordinating delegation costs more
than doing it.

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

Never delegate (stays inline — no second lane to fall back to): domain
logic, business rules, architecture decisions, complex refactors requiring
full codebase context, build/type errors needing multi-step diagnosis,
anything where the WHY lives in this conversation.

Rules:
- Launch delegations in the background and keep working — never idle waiting.
- WIP cap: 3–5 concurrent delegations. Kill-switch: 3 failed iterations on
  the same task → stop retrying, bring it inline (there's no other lane to
  hand it to instead).
- Quota fallback: Copilot hits quota (429/5xx/"rate limit") → nothing to
  fail over to. Stop auto-delegating for the rest of the session, do the
  task inline, and say so once.
- Copilot-specific bug: resuming a session after a rate-limit hit can hang.
  On "rate limit" in output + ~10s of silence: kill the process, don't wait,
  relaunch fresh without `--continue`.
```

If you later add a reasoning delegate (Codex or otherwise), switch to the
triad block above and change the quota rule back to failing over between
lanes instead of dropping straight to inline.
