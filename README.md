# copilot-plugin-cc

Delegate mechanical coding tasks from [Claude Code](https://claude.com/claude-code)
to [GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli)
in non-interactive autonomous mode.

Claude Code stays the orchestrator — it writes the domain logic, defines each
subtask contract, and reviews the diffs. Copilot CLI absorbs the purely
mechanical work (boilerplate, renames, dead-code cleanup, simple specs, DTO
mapping, PR descriptions) in the background, so your Claude context and tokens
go to the work only Claude can do.

Inspired by the structure of [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc).
**Not affiliated with GitHub, Microsoft, OpenAI, or Anthropic.**

> Lea esto en español: [README.es.md](README.es.md)

## Requirements

- Claude Code
- A GitHub Copilot subscription
- GitHub Copilot CLI ≥ **1.0.67** (`npm install -g @github/copilot`)

## Install

In Claude Code:

```
/plugin marketplace add santiquiroz/copilot-plugin-cc
/plugin install copilot@copilot-plugin-cc
```

Then verify your environment:

```
/copilot:setup
```

## Usage

Explicit delegation:

```
/copilot:rescue remove all unused imports under src/ and fix the import order
/copilot:rescue --background generate boilerplate test specs for src/services/user-mapper.ts
/copilot:rescue --model claude-sonnet-5 rename WidgetFactory to WidgetBuilder across the repo
```

Proactive delegation: the `copilot-rescue` agent describes itself so Claude
Code picks it for mechanical tasks on its own. To wire it into your own
delegation rules, paste the block from
[docs/claude-md-snippet.md](docs/claude-md-snippet.md) into your `CLAUDE.md`.

Full orchestration patterns — the Codex/Copilot/inline split, the parallel
pattern, WIP caps, and the quota fallback chain — live in
[docs/delegation-guide.md](docs/delegation-guide.md).

## Safety model

Every forwarded task runs Copilot CLI with a scoped flag set:

```
copilot -p "<task>" -s \
  --allow-tool='shell(git:*)' --allow-tool=write \
  --deny-tool='shell(rm)' --deny-tool='shell(git push)' --deny-tool='shell(git reset)'
```

Deny rules win over allow rules — even under `--allow-all` — so a mechanical
task can write files and use local git, but can never delete files, push, or
reset shared state. `--allow-all` is only used when a task genuinely needs a
tool outside git/write and the user confirmed it.

## Known upstream issues this plugin works around

| Issue | Workaround baked in |
|---|---|
| Autopilot infinite loop on externally-blocked tasks ([copilot-cli#2969](https://github.com/github/copilot-cli/issues/2969)) | `--autopilot` avoided for bounded tasks; when used, `--max-autopilot-continues <N>` is always pinned explicitly |
| Resume after a rate-limit hit can hang | On "rate limit" output + ~10s silence: kill the process and report, never wait; relaunch fresh without `--continue` |

## What's in the plugin

| Piece | Purpose |
|---|---|
| `agents/copilot-rescue.md` | Thin forwarder subagent — one `copilot -p` call, output returned verbatim |
| `/copilot:rescue` | Delegate a task explicitly (`--background`, `--wait`, `--model <name>`) |
| `/copilot:setup` | Verify CLI install, version floor, auth, and model pinning options |
| `docs/delegation-guide.md` | Full multi-agent orchestration guide |
| `docs/claude-md-snippet.md` | Ready-to-paste CLAUDE.md block |

## License

[MIT](LICENSE)
