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
- GitHub Copilot CLI ≥ **1.0.83** recommended (`npm install -g @github/copilot`); 1.0.67 remains the minimum for `--model`

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
/copilot:rescue --credits 50 generate boilerplate test specs for src/services/user-mapper.ts
/copilot:rescue --allow-shell dotnet add null-guard tests to OrderMapperTests.cs and make sure dotnet test passes
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
copilot -p "$(cat <<'COPILOT_TASK'
<task>
COPILOT_TASK
)" -s --no-ask-user --max-ai-credits 30 \
  --allow-tool='shell(git:*)' --allow-tool=write \
  --deny-tool='shell(rm)' --deny-tool='shell(git push)' \
  --deny-tool='shell(git reset)' --deny-tool='shell(git clean)' \
  --deny-tool='shell(git checkout)' --deny-tool='shell(git restore)' \
  --deny-tool='shell(git switch)' --deny-tool='shell(git rm)' \
  --deny-tool='shell(git stash)' --deny-tool='shell(git worktree)' \
  --deny-tool='shell(git submodule)' --deny-tool='shell(git config)'
```

The task goes through a quoted heredoc, never inline in double quotes, so
backticks, `$` and quotes in it reach Copilot literally instead of being run or
mangled by the shell. The default cap is 30 AI credits (the CLI minimum),
`--no-ask-user` prevents the agent from stalling for human input, and deny
rules win over allow rules — so while only git is allowed, a mechanical task
can write files and use local git but cannot directly run `rm` or the
destructive git subcommands (`push`, `reset`, `clean`, `checkout`, `restore`,
`switch`, `rm`, `stash`, `worktree`, `submodule`, `config`). Matching is per
first-level subcommand, and git itself can run code through aliases
(`git -c alias.x='!cmd' x`) or hooks Copilot writes, so this is a guardrail,
not a sandbox: review `git status`, `git log` and `git reflog` after every run.
`--credits <N>` overrides the cap.

Only `git` may run by default; in `-p` mode any other shell command is denied
without a prompt. When a task must build, test or lint, pass
`--allow-shell <tool>[,<tool>]` — or ask for the command explicitly in the task
(e.g. "run `dotnet test`"): build-toolchain tools (`dotnet`, `npm`, `ng`,
`pytest`, …) are then allowed automatically; interpreters such as `node` or
`python` need an explicit `--allow-shell`. Each tool becomes one
`--allow-tool='shell(<tool>:*)'`. Command shells (`powershell`, `cmd`, `bash`…),
command runners and privilege tools (`env`, `xargs`, `sudo`…), deletion commands
and network/remote/cloud CLIs (`gh`, `ssh`, `curl`, `az`…) are refused, and
`--allow-all` is never used.

Deny rules only match commands Copilot runs directly. Once any shell tool is
allowed — via `--allow-shell` or auto-allow, which triggers on task wording
alone and does not check whether the repo is trusted — Copilot can write a
script, npm script, Makefile recipe or MSBuild target and run it through that
tool, and that code can delete files, push or use the network without hitting a
deny rule. Treat any shell allowance as full code execution with your
credentials, and review `git status`, `git log` and `git reflog` after the run.
Private package feeds still need credentials in Copilot's environment — a 401
on restore is auth, not permissions.

## Known upstream issues this plugin works around

| Issue | Workaround baked in |
|---|---|
| Autopilot infinite loop on externally-blocked tasks ([copilot-cli#2969](https://github.com/github/copilot-cli/issues/2969)) | `--autopilot` avoided for bounded tasks; when used, `--max-autopilot-continues <N>` is always pinned explicitly |
| Resume after a rate-limit hit can hang | On "rate limit" output + ~10s silence: kill the process and report, never wait; relaunch fresh without `--continue` |
| CLI older than 1.0.83 rejects safety/credit flags | Retry once without `--no-ask-user` and `--max-ai-credits`, then update Copilot CLI |
| Copilot CLI 1.0.83 rejects `--max-ai-credits` below 30 and `--effort` on the default Auto model | Default cap is 30 (lower `--credits` raised to 30); `--effort` is forwarded only with a pinned non-`auto` `--model` |

## What's in the plugin

| Piece | Purpose |
|---|---|
| `agents/copilot-rescue.md` | Thin forwarder subagent — one `copilot -p` call, output returned verbatim |
| `/copilot:rescue` | Delegate a task explicitly (`--background`, `--wait`, `--model <name>`, `--credits <N>`, `--effort <level>`, `--allow-shell <tool>`) |
| `/copilot:setup` | Verify CLI install, version floor, auth, and model pinning options |
| `docs/delegation-guide.md` | Full multi-agent orchestration guide |
| `docs/claude-md-snippet.md` | Ready-to-paste CLAUDE.md block |
| `.codex-plugin/plugin.json` | Codex CLI plugin manifest (experimental native install) |
| `skills/copilot-rescue/SKILL.md` | Codex skill — same forwarder logic, Codex runs the shell call itself (no subagent layer) |
| `docs/agents-md-snippet.md` | Ready-to-paste `AGENTS.md` block for Codex users |

## Codex CLI

The same idea, for [Codex CLI](https://developers.openai.com/codex/): delegate
mechanical work to GitHub Copilot CLI, so Codex's reasoning stays on the work
only it can do. Ships as a Codex **skill** (`skills/copilot-rescue/SKILL.md`)
instead of a subagent — Codex runs the forwarded `copilot -p ...` command
itself, since Codex has no separate subagent/Task layer.

### Requirements

- Codex CLI
- A GitHub Copilot subscription
- GitHub Copilot CLI ≥ **1.0.83** recommended (`npm install -g @github/copilot`); 1.0.67 remains the minimum for `--model`

### Install

**Manual (confirmed working):**

```bash
mkdir -p ~/.agents/skills
cp -r skills/copilot-rescue ~/.agents/skills/copilot-rescue
```

Or repo-scoped only: copy into `<your-repo>/.agents/skills/copilot-rescue/` instead.

**Native plugin marketplace (experimental — Codex's plugin system is new; please
open an issue if the paths below don't match your Codex CLI version):**

```
codex plugin marketplace add santiquiroz/copilot-plugin-cc
```

Then open the plugin browser (`/plugins` inside Codex) and install `copilot`.

### Usage

Codex matches skills implicitly by their `description`, or you can invoke
explicitly. Ask for a mechanical task and Codex should pick `copilot-rescue`
on its own; to wire proactive delegation into your own instructions, paste the
block from [docs/agents-md-snippet.md](docs/agents-md-snippet.md) into your
`AGENTS.md`.

### Safety model

Same as the Claude Code side — see [Safety model](#safety-model) above. The
skill runs Copilot CLI with the identical scoped allow/deny flag set.

## License

[MIT](LICENSE)
