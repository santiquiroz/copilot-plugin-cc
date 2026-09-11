---
name: copilot-rescue
description: Delegate a mechanical, zero-domain-context coding task to GitHub Copilot CLI in non-interactive mode — boilerplate, mechanical renames, dead code cleanup, simple CRUD/mapping specs, DTO-to-interface generation, PR descriptions for self-explanatory commits. Do not use for domain logic, architecture decisions, or multi-step debugging.
---

Forward the requested task to GitHub Copilot CLI with one shell command. Do not do the task yourself once Copilot is invoked — return its output.

## Command

Never put the task inside double quotes: the shell would expand backticks, `$` and quotes in it (bash runs backticked commands itself, outside every deny rule; PowerShell treats the backtick as an escape character). Pass it through a literal heredoc / here-string.

bash:

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

PowerShell 7.3+ (`pwsh`; the closing `'@` must start at column 0). Windows PowerShell 5.1 mangles embedded double quotes when calling native programs — there, use the bash form through Git Bash or run this under `pwsh`:

```
$task = @'
<task>
'@
copilot -p $task -s --no-ask-user --max-ai-credits 30 --allow-tool='shell(git:*)' --allow-tool=write --deny-tool='shell(rm)' --deny-tool='shell(git push)' --deny-tool='shell(git reset)' --deny-tool='shell(git clean)' --deny-tool='shell(git checkout)' --deny-tool='shell(git restore)' --deny-tool='shell(git switch)' --deny-tool='shell(git rm)' --deny-tool='shell(git stash)' --deny-tool='shell(git worktree)' --deny-tool='shell(git submodule)' --deny-tool='shell(git config)'
```

Deny rules win over allow rules for commands Copilot runs directly — this keeps the run non-interactive-capable while blocking the destructive/shared-state commands a mechanical task never needs. Matching is per first-level git subcommand, so it is a guardrail, not a sandbox. The default credit cap is 30 (the CLI minimum); `--credits <N>` overrides it. Caller-supplied `--effort <level>` is forwarded only with a pinned `--model` other than `auto` — Auto rejects effort.

## Rules

- Preserve the user's task text verbatim in `-p`, minus the runtime flags (`--model`, `--credits`, `--effort`, `--allow-shell`). Do not add commentary or hedging.
- `-s` / `--silent` strips usage-stats noise so returned stdout is clean.
- Add `--model <name>` only if the user named a model; otherwise omit (Auto-selection carries a billing discount on routine work).
- Add `--max-ai-credits <N>` for caller-supplied `--credits <N>` (minimum 30), or use the default of 30.
- Add `--effort <level>` only when the caller also pinned a `--model` other than `auto`; otherwise drop it.
- Add `--add-dir <path>` if the task is scoped outside the current working directory; add `-C <dir>` only if it explicitly targets a different working directory.
- Run the command synchronously — wait for it to finish, don't background it.
- If the user says "continue"/"keep going"/"resume" prior Copilot work here, add `--continue` instead of starting fresh.
- Do not inspect the repo, grep, or do follow-up work beyond the one forwarded command — Copilot does the task, you relay its output.
- If Copilot rejects `--no-ask-user` or `--max-ai-credits` as unknown, rerun once without both flags and tell the caller to update the CLI.
- Only `git` may run by default; in `-p` mode any shell command not matched by an `--allow-tool` pattern is denied without a prompt, so build/test/lint steps fail unless allowed. Add one `--allow-tool='shell(<tool>:*)'` per tool when the caller supplies `--allow-shell <tool>[,<tool>...]` (remove it from the task text), or when the task explicitly asks to run a command from the toolchain allowlist: `dotnet`, `npm`, `ng`, `yarn`, `pnpm`, `pytest`, `mvn`, `gradle`, `go`, `cargo`, `make`. Interpreters and package runners (`node`, `python`, `py`, `npx`, `pip`, `perl`, `ruby`, …) are allowed only through an explicit `--allow-shell`.
- Never allow — the listed tools and anything else in the same category — command shells (`powershell`, `pwsh`, `cmd`, `bash`, `sh`, `zsh`, `fish`, `wsl`), command runners and privilege tools (`env`, `xargs`, `sudo`, `runas`, `Start-Process`), deletion commands (`del`, `rd`, `rmdir`, `Remove-Item`), network/remote/cloud/cluster CLIs (`curl`, `wget`, `Invoke-WebRequest`, `ssh`, `scp`, `rsync`, `gh`, `az`, `aws`, `gcloud`, `kubectl`, `docker`, `helm`) or extra `git` scope, even on request (the interpreters and package runners above are not in the command-runner category) — refuse with `refused --allow-shell <tool>: not permitted`. Never use `--allow-all`; keep every `--deny-tool` flag. Deny rules match only commands Copilot runs directly, not processes an allowed tool spawns — any shell allowance is full code execution, so after every run add one line telling the caller to review `git status`, `git log` and `git reflog` (do not run them yourself) — even the default git allowance can run code through git aliases or hooks.
- Only use `--autopilot` for genuinely open-ended multi-step tasks, and always pin `--max-autopilot-continues <N>` (e.g. 8) when you do — Copilot CLI has a known infinite-loop bug on externally-blocked tasks under autopilot (github/copilot-cli#2969).

## Known issues to work around

- **Autopilot infinite loop** (copilot-cli#2969): avoid `--autopilot` for bounded tasks; pin `--max-autopilot-continues` when used.
- **Resume-after-rate-limit hang**: if output contains "rate limit" (case-insensitive) and then goes silent ~10s, kill the process and report failure — do not wait, do not `--continue` a rate-limited session.

## Quota / failure handling

If the command fails (non-zero exit, auth error, or a rate-limit message), report the error text verbatim instead of retrying silently — the caller decides whether to fall back to another approach or take the task over directly. To probe auth/health first: `copilot --version`, then `copilot -p "Reply with exactly one word: ready" -s --no-ask-user --deny-tool=shell --deny-tool=write`.

## Output

Return Copilot's stdout as-is. Do not paraphrase or summarize it.
