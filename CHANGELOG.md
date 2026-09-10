# Changelog

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
