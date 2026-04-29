# Changelog

## 0.1.0 — 2026-04-29

Initial release. Migrated from `ai-oil-pollution-analysis` project-level setup to a standalone Claude Code plugin marketplace.

### Added

- `code` plugin containing three skills:
  - `code-feat` (Tier 3 SDD pipeline)
  - `code-fix` (Tier 2 lightweight pipeline)
  - `code-review` (standalone Codex-backed review)
- Slash commands `/code:feat`, `/code:fix`, `/code:review` (trampolines into the corresponding skills).
- `docs/ai-development-pipeline.md` — full methodology documentation.

### Changed (vs. the original project-level skills)

- Codex integration no longer relies on `${CLAUDE_PLUGIN_ROOT}` (which would resolve to this plugin's root, not openai-codex's). Both `code-feat` and `code-review` now auto-discover `codex-companion.mjs` under `~/.claude/plugins/cache/openai-codex/codex/*/scripts/`, overridable via `CODEX_COMPANION`.
- Removed hardcoded openai-codex path/version from `code-review` Step 3.
- Generalized project-specific review checks: removed `.glass-card`, `14px / 24px+` examples; replaced with "follow project design system conventions" and "rules defined in project CLAUDE.md".
- `code-fix` Spec impact check now references only `openspec/specs/` and "design-document locations defined in project CLAUDE.md", removing the project-specific `docs/plans/` paths.
