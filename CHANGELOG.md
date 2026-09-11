# Changelog

## 0.3.1 — 2026-09-10

- Fix build/test tasks failing with a silent permission denial: in `-p`
  (non-interactive) mode every shell command not matched by an `--allow-tool`
  pattern (i.e. everything but git) was denied without a prompt, and the
  opt-in tool allowance was vague enough that callers never triggered it.
- Add the `--allow-shell <tool>[,<tool>]` runtime flag, auto-allow build
  toolchains (`dotnet`, `npm`, `ng`, `pytest`…) when the task explicitly asks
  to run them (interpreters such as `node`/`python` need the explicit flag),
  and refuse command shells, command runners and privilege tools, deletion
  commands and network/remote/cloud CLIs (including `gh` and `ssh`) by name
  and by category.
- Remove the contradictory `--allow-all` fallback from the forwarder.
- Fix every forwarded call failing on Copilot CLI 1.0.83: the default
  `--max-ai-credits 10` is below the CLI's 30-credit minimum (default is now
  30; lower `--credits` values are raised to 30), and `--effort` is rejected
  by Auto (now forwarded only with a pinned `--model` other than `auto`; the
  automatic `--effort low` for tasks marked mechanical is gone).
- Pass the task through a quoted heredoc instead of double quotes: backticks
  in the task (which this release encourages, e.g. "run `dotnet test`") were
  executed by the forwarder's own shell, outside every deny rule, and `$` and
  quotes were mangled. The Codex skill gets bash and PowerShell forms.
- Also deny `git restore`, `git switch`, `git rm`, `git stash`,
  `git worktree`, `git submodule` and `git config`, and stop claiming the flag
  set "never" deletes files: deny rules only match direct commands, so any
  shell allowance is now documented as full code execution, and callers are
  told to review `git status`, `git log` and `git reflog` after every run
  (git alone can run code through aliases or hooks).
- The agent description now tells callers that only `git` runs unless
  `--allow-shell` or an explicit command is given; the CLAUDE.md and
  AGENTS.md snippets say the same.

## 0.3.0 — 2026-09-10

- Align Copilot forwarding with CLI 1.0.83: disable interactive questions,
  cap default spend at 10 AI credits, support caller-selected effort, and
  deny `rm`, `git push`, `git reset`, `git clean`, and `git checkout`.
- Add `--credits <N>` and `--effort <level>` runtime flags, an older-CLI
  compatibility retry, and explicit opt-in tool allowances.

## 0.2.0 — 2026-07-22

- Add Codex CLI compatibility: `copilot-rescue` ships as a Codex skill
  (`skills/copilot-rescue/SKILL.md`) with a `.codex-plugin/plugin.json`
  manifest for native `codex plugin marketplace add` install (experimental —
  Codex's plugin system is new, please open an issue if paths don't match
  your Codex CLI version), plus a manual-copy install path (confirmed
  working) and an `AGENTS.md` delegation snippet
  (`docs/agents-md-snippet.md`).

## 0.1.0 — 2026-07-03

Initial release.

- `copilot-rescue` subagent: thin forwarder to GitHub Copilot CLI with a
  hardened scoped allow/deny flag set for non-interactive autonomous runs.
- `/copilot:rescue` command: delegate a mechanical task from any session.
- `/copilot:setup` command: verify Copilot CLI install, version (≥ 1.0.67),
  authentication, and model pinning options.
- Delegation guide + ready-to-paste CLAUDE.md snippet.
