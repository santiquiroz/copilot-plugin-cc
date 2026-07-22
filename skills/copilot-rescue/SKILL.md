---
name: copilot-rescue
description: Delegate a mechanical, zero-domain-context coding task to GitHub Copilot CLI in non-interactive mode — boilerplate, mechanical renames, dead code cleanup, simple CRUD/mapping specs, DTO-to-interface generation, PR descriptions for self-explanatory commits. Do not use for domain logic, architecture decisions, or multi-step debugging.
---

Forward the requested task to GitHub Copilot CLI with one shell command. Do not do the task yourself once Copilot is invoked — return its output.

## Command

```
copilot -p "<task>" -s \
  --allow-tool='shell(git:*)' --allow-tool=write \
  --deny-tool='shell(rm)' --deny-tool='shell(git push)' --deny-tool='shell(git reset)'
```

Deny rules win over allow rules, even under `--allow-all` — this keeps the run non-interactive-capable while blocking destructive/shared-state commands a mechanical task never needs.

## Rules

- Preserve the user's task text verbatim in `-p`. Do not add commentary or hedging.
- `-s` / `--silent` strips usage-stats noise so returned stdout is clean.
- Add `--model <name>` only if the user named a model; otherwise omit (Auto-selection carries a billing discount on routine work).
- Add `--add-dir <path>` if the task is scoped outside the current working directory; add `-C <dir>` only if it explicitly targets a different working directory.
- Run the command synchronously — wait for it to finish, don't background it.
- If the user says "continue"/"keep going"/"resume" prior Copilot work here, add `--continue` instead of starting fresh.
- Do not inspect the repo, grep, or do follow-up work beyond the one forwarded command — Copilot does the task, you relay its output.
- Only use `--autopilot` for genuinely open-ended multi-step tasks, and always pin `--max-autopilot-continues <N>` (e.g. 8) when you do — Copilot CLI has a known infinite-loop bug on externally-blocked tasks under autopilot (github/copilot-cli#2969).

## Known issues to work around

- **Autopilot infinite loop** (copilot-cli#2969): avoid `--autopilot` for bounded tasks; pin `--max-autopilot-continues` when used.
- **Resume-after-rate-limit hang**: if output contains "rate limit" (case-insensitive) and then goes silent ~10s, kill the process and report failure — do not wait, do not `--continue` a rate-limited session.

## Quota / failure handling

If the command fails (non-zero exit, auth error, or a rate-limit message), report the error text verbatim instead of retrying silently — the caller decides whether to fall back to another approach or take the task over directly. To probe auth/health first: `copilot --version`, then `copilot -p "Reply with exactly one word: ready" -s --deny-tool=shell --deny-tool=write`.

## Output

Return Copilot's stdout as-is. Do not paraphrase or summarize it.
