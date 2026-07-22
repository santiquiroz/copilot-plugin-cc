# AGENTS.md snippet (Codex)

Paste into your repo's `AGENTS.md` (or `~/.codex/AGENTS.md` for a global default) to make Codex delegate mechanical work to GitHub Copilot CLI proactively, via the `copilot-rescue` skill from this plugin.

```markdown
# Copilot CLI delegation (copilot-rescue skill)

Copilot owns pure-mechanical, zero-domain-context work. Use the `copilot-rescue`
skill proactively — no user prompt needed — for:

- Simple test specs (CRUD/mapping, no branching logic)
- Mechanical renames across 3+ files
- Dead code removal / unused imports / debug-statement cleanup
- Interface generation from DTOs/entities (pure field mapping)
- Component boilerplate shells
- Identical config blocks applied to N similar files
- PR descriptions when commits are self-explanatory

Never delegate: domain logic, business rules, architecture decisions, complex
refactors requiring full codebase context, anything where the WHY lives in
this conversation.

On a Copilot CLI "rate limit" message: stop, don't retry automatically, don't
resume with `--continue` on a rate-limited run — start fresh or fall back to
doing the task directly.
```

Requires GitHub Copilot CLI ≥ 1.0.67 installed and authenticated (`copilot --version`, then `copilot -p "..." -s`). See the main [README](../README.md) for install steps.
