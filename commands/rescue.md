---
description: Delegate a mechanical, zero-domain-context task to the Copilot rescue subagent
argument-hint: "[--background|--wait] [--model <name>] [--credits <N>] [--effort <level>] [--allow-shell <tool>[,<tool>]] [the mechanical task Copilot should perform]"
allowed-tools: AskUserQuestion, Agent
---

Invoke the `copilot:copilot-rescue` subagent via the `Agent` tool (`subagent_type: "copilot:copilot-rescue"`), forwarding the raw user request as the prompt.
`copilot:copilot-rescue` is a subagent, not a skill — do not call it via the `Skill` tool. This command runs inline so the `Agent` tool stays in scope.
The final user-visible response must be Copilot's output verbatim (plus the one review line described below).

Raw user request:
$ARGUMENTS

Execution mode:

- If the request includes `--background`, run the subagent in the background and continue other work; relay the result when it completes.
- If the request includes `--wait`, run the subagent in the foreground.
- If neither flag is present, default to foreground.
- `--background` and `--wait` are execution flags for Claude Code. Do not forward them in the prompt, and do not treat them as part of the natural-language task text.
- `--model <name>` is a runtime-selection flag. Preserve it in the forwarded prompt (the subagent maps it to copilot's `--model` flag), but do not treat it as part of the natural-language task text.
- `--credits <N>` is a runtime-selection flag. It overrides the default 30-credit cap (the Copilot CLI minimum; lower values are raised to 30) and is forwarded as `--max-ai-credits <N>`.
- `--effort <level>` is a runtime-selection flag forwarded as `--effort <level>` only together with a `--model <name>` other than `auto`; Auto (implicit or `--model auto`) rejects reasoning-effort configuration, so otherwise it is dropped.
- `--allow-shell <tool>[,<tool>...]` is a runtime-selection flag. Preserve it in the forwarded prompt; the subagent maps each tool to `--allow-tool='shell(<tool>:*)'` so Copilot can build, test or lint. Without it (or an explicit build-toolchain command like "run `dotnet test`" in the task — interpreters and package runners such as `node`, `python` or `npx` always need `--allow-shell`), every non-git shell command is denied without a prompt in `-p` mode. Command shells, command runners and privilege tools (`env`, `xargs`, `sudo`…), deletion commands and network/remote/cloud CLIs are refused. Any allowance is full code execution: Copilot can write a script and run it through the allowed tool, beyond the reach of the deny rules.

Operating rules:

- The subagent is a thin forwarder only. It uses one `Bash` call to invoke `copilot -p ...` with a scoped allow/deny flag set and returns that command's stdout as-is.
- Return the Copilot output verbatim to the user. Do not paraphrase, summarize, rewrite, or add commentary before or after it. Exception: always add one line after the output telling the user to review `git status`, `git log` and `git reflog` — even the default git allowance can run code through git aliases or hooks, and any extra shell allowance is full code execution.
- Do not ask the subagent to inspect files, monitor progress, summarize output, or do follow-up work of its own.
- If the returned output shows Copilot is not installed or not authenticated, tell the user to run `/copilot:setup`.
- If the returned output shows a rate limit, report it and suggest retrying later or falling back to another delegate — do not retry automatically.
- If the user did not supply a task, ask what mechanical task Copilot should perform.
