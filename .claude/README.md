# Claude Code configuration — KIDS/RAFAELA_JOGO_MEMORIA

Stack: **Laravel**.

## Files
- `settings.json` — the ACTIVE profile.
- `settings.local.json` — local override (gitignored), takes precedence over `settings.json`.

## Model and effort
- **This repository does not choose the model** (repodocs ADR-027). `settings.json`
  carries no `model` and no `fallbackModel`, and nothing in `env` steers one — no
  `ANTHROPIC_MODEL`, no `ANTHROPIC_DEFAULT_*_MODEL`, no `CLAUDE_CODE_SUBAGENT_MODEL`.
- The model is the user's choice, made per session with `/model`; a subagent inherits
  the session's model. There are no stand-by profiles to copy over `settings.json`.
- Effort `max` via the `CLAUDE_CODE_EFFORT_LEVEL` env var (the `effortLevel` field
  only accepts low/medium/high/xhigh).

## Permissions
- `defaultMode: plan`; security denies (rm -rf, force push, reset --hard, clean -fd, curl|sh).
- **git push allowed** (in `allow`).
