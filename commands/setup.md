---
description: Check whether GitHub Copilot CLI is installed, authenticated, and ready for delegation
argument-hint: ""
allowed-tools: Bash(copilot:*), Bash(npm:*), AskUserQuestion
---

Run these checks in order and report a single consolidated status at the end.

Step 1 — Installed?

Run:

```bash
copilot --version
```

- If the command is not found and `npm` is available: use `AskUserQuestion` exactly once with two options — `Install GitHub Copilot CLI (Recommended)` and `Skip for now`. If the user chooses install, run `npm install -g @github/copilot`, then rerun `copilot --version`.
- If the command is not found and `npm` is unavailable: report the official install docs (https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli) and stop.

Step 2 — Version gate

- If the reported version is lower than 1.0.67: warn that multi-model selection (`--model`) is unsupported on this version and recommend `npm update -g @github/copilot`. Continue with the remaining checks.

Step 3 — Authentication probe

Run:

```bash
copilot -p "Reply with exactly one word: ready" -s --deny-tool=shell --deny-tool=write
```

- Output contains `ready` → authenticated and working.
- Output shows an authentication error → instruct the user to either run `copilot` interactively and use `/login`, or set the `COPILOT_GITHUB_TOKEN` environment variable with a token that has Copilot access. NEVER print or echo the token value.
- Output contains "rate limit" (case-insensitive) → report that the Copilot quota is currently exhausted; delegation is unavailable until it resets.

Step 4 — Consolidated report

Summarize in one short block: install state, version (and whether it meets the 1.0.67 floor), auth state, and model options — pin with `--model <name>` or the `COPILOT_MODEL` environment variable for deterministic behavior; omit (or use `--model auto`) to allow Auto-selection, which carries a billing discount on routine mechanical work.
