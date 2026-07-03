---
description: Delegate a mechanical, zero-domain-context task to the Copilot rescue subagent
argument-hint: "[--background|--wait] [--model <name>] [the mechanical task Copilot should perform]"
allowed-tools: AskUserQuestion, Agent
---

Invoke the `copilot:copilot-rescue` subagent via the `Agent` tool (`subagent_type: "copilot:copilot-rescue"`), forwarding the raw user request as the prompt.
`copilot:copilot-rescue` is a subagent, not a skill — do not call it via the `Skill` tool. This command runs inline so the `Agent` tool stays in scope.
The final user-visible response must be Copilot's output verbatim.

Raw user request:
$ARGUMENTS

Execution mode:

- If the request includes `--background`, run the subagent in the background and continue other work; relay the result when it completes.
- If the request includes `--wait`, run the subagent in the foreground.
- If neither flag is present, default to foreground.
- `--background` and `--wait` are execution flags for Claude Code. Do not forward them in the prompt, and do not treat them as part of the natural-language task text.
- `--model <name>` is a runtime-selection flag. Preserve it in the forwarded prompt (the subagent maps it to copilot's `--model` flag), but do not treat it as part of the natural-language task text.

Operating rules:

- The subagent is a thin forwarder only. It uses one `Bash` call to invoke `copilot -p ...` with a scoped allow/deny flag set and returns that command's stdout as-is.
- Return the Copilot output verbatim to the user. Do not paraphrase, summarize, rewrite, or add commentary before or after it.
- Do not ask the subagent to inspect files, monitor progress, summarize output, or do follow-up work of its own.
- If the returned output shows Copilot is not installed or not authenticated, tell the user to run `/copilot:setup`.
- If the returned output shows a rate limit, report it and suggest retrying later or falling back to another delegate — do not retry automatically.
- If the user did not supply a task, ask what mechanical task Copilot should perform.
