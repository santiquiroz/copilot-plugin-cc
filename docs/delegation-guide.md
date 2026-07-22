# Multi-Agent Delegation Guide

How to make Claude Code delegate work to GitHub Copilot CLI (this plugin) and,
optionally, to a reasoning-capable delegate such as the codex plugin's
`codex-rescue` — so the main Claude thread stays focused on the work only it
can do.

## Which setup do you have?

This guide describes the full 3-lane setup: Claude Code orchestrates, a
reasoning-capable delegate (Codex, or any other CLI-drivable agent that can
reason about a codebase) takes reasoning-heavy work, and Copilot CLI (this
plugin) takes mechanical work. You don't need Codex specifically — the
"reasoning delegate" lane is whatever second agent you have wired in.

**No second delegate at all?** Only the Copilot lane exists. Mechanical work
still delegates out; everything this guide calls "reasoning delegate" work
(build-fixing, multi-file refactors, logic-bearing code) stays inline with
Claude instead of failing over to another lane. Use the no-reasoning-delegate
variant in [docs/claude-md-snippet.md](claude-md-snippet.md) and skip the
"other lane" steps in the sections below — a quota-exhausted Copilot just
means "stop delegating, do it inline," since there's nothing to fail over to.

## The core split

| Lane | Owns | Examples |
|---|---|---|
| **Reasoning delegate** (e.g. Codex) | Anything needing real reasoning, multi-step build fixing, or codebase-wide context | Build errors after a first failed fix, logic-bearing handlers/services, multi-file refactors that change control flow, complex test specs |
| **Copilot (this plugin)** | Purely mechanical, zero-domain-context work | Simple CRUD/mapping test specs, mechanical renames across 3+ files, dead code / unused import cleanup, interface generation from DTOs, component boilerplate shells, identical config blocks across N files, PR descriptions for self-explanatory commits |
| **Keep inline (never delegate)** | Tasks where the WHY lives in your conversation | Domain logic and business rules, architecture and feature design, refactors requiring full codebase context |

Rule of thumb: if the delegate needs to understand *why*, it is not a Copilot
task. If it is purely mechanical shape/pattern work, it is.

## The default-delegate discipline

- Before writing any code, decide the lane explicitly: reasoning delegate,
  Copilot, or inline. Inline is only for tasks on the never-delegate list or
  trivial edits (<5 lines, 1 file) where coordinating a delegation costs more
  than doing it.
- The main thread acts as orchestrator and reviewer: define the subtask
  contract, launch the delegation in the background, review the diff when it
  returns. Never idle while a delegation runs — continue with the next
  subtask.

## The parallel pattern

```
Claude: writes SomeHandler (domain logic — inline, never delegated)
  → immediately launches reasoning delegate in background: "fix build errors in related module"
  → immediately launches /copilot:rescue --background "generate boilerplate tests for SomeHandler"
Claude: continues with the next task while both run
```

- WIP cap: 3–5 concurrent background delegations. Beyond that, coordination
  overhead and unreviewed compounding mistakes outweigh the parallelism gain.
- Kill-switch: after 3 stuck/failed iterations on the same delegated task,
  stop retrying that delegate. Hand it to the other lane once, or bring it
  back inline. Do not loop indefinitely.

## Safety model

The `copilot-rescue` agent always runs Copilot CLI with a scoped flag set:

```
copilot -p "<task>" -s \
  --allow-tool='shell(git:*)' --allow-tool=write \
  --deny-tool='shell(rm)' --deny-tool='shell(git push)' --deny-tool='shell(git reset)'
```

- Deny rules always win over allow rules — even under `--allow-all`. The
  denies block destructive and shared-state commands a mechanical task never
  needs (`rm`, `git push`, `git reset`) while keeping the run fully
  non-interactive.
- `--autopilot` is avoided for bounded tasks. When a task is genuinely
  open-ended, `--max-autopilot-continues <N>` is always pinned explicitly:
  Copilot CLI has a known infinite-loop bug on externally-blocked tasks
  (upstream issue #2969).

## Quota fallback chain

Delegates hit rate limits and quota caps. Never let a quota error silently
kill a task.

**Detection:** scan delegate output for `rate limit`, `hit a rate limit`
(Copilot's literal phrasing), `quota`, `429`, `Too Many Requests`,
`usage limit`, `insufficient_quota`, or `403` paired with a billing message.

**429 vs 5xx:** an error carrying a wait-time/retry hint (e.g. `retry-after`)
is a soft cap — wait that long and retry the SAME delegate once before failing
over. A server/overload error (5xx, "overloaded", no retry hint) → fail over
to the other lane immediately.

**Copilot-specific:** resuming a session after a rate-limit hit can hang
(documented unresolved CLI bug). Kill and relaunch fresh — no `--continue` —
rather than waiting on a stuck process.

**The chain:**

1. Primary delegate (per the split table) hits quota.
2. If the task is mechanical-compatible → retry once on the other lane
   (Copilot quota → try the reasoning delegate; and vice versa).
3. If the task needs reasoning and only one delegate fits, or step 2 also hits
   quota → the main thread does the task inline. Do not loop retrying.

Tell the user when a fallback happened — which delegate failed, which one
picked it up, or that the main thread took over. One line, no drama.

If BOTH lanes are quota-exhausted in the same session: stop auto-delegating
for the rest of the session, handle everything inline, and mention this once.

## Model diversity as a bounded second lane

Copilot CLI (≥ 1.0.67) can run multiple frontier models — pin with
`--model <name>` or the `COPILOT_MODEL` env var for deterministic behavior,
or omit for the Auto-selection billing discount on routine mechanical work.
For a single well-scoped, non-architectural task where a second independent
implementation opinion is useful, running `/copilot:rescue --model <name>` as
a second lane is reasonable. This does NOT relax the never-delegate list; it
only widens what the mechanical lane can attempt.
