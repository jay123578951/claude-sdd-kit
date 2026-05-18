# Changelog

## 0.4.0 — 2026-05-18

Coder agent 改為條件式 model 選擇：預設維持 sonnet 以節省額度，僅在先驗上需要深度推理或 retry 卡關時升 opus。Pipeline 的 Tester + Opus Reviewer + retry 結構已能接住 sonnet 第一版的表層瑕疵，故不全面切換。

### Changed

- `code-feat` 新增 Step 3「Coder Model 升級判定」：架構變更 / 大型重構、安全敏感路徑、設計決策密集任一成立時 `{coderModel}` 升 opus，否則維持 sonnet。Step 4 派發改用 `{coderModel}` 變數。
- `code-feat` Retry 迴路：Reviewer counter ≥ 2 時，除既有的強制 adversarial 外，`{coderModel}` 一併強制升 opus 且不再降回。
- `code-fix` Coder 改為 sonnet（預設）/ opus：因 Tier 2 依定義無設計決策，僅保留「安全敏感路徑」首次升級條件，加上「Coder ↔ Tester retry 第 2 輪起升 opus」的動態升級。

## 0.3.0 — 2026-05-13

Replaced the Codex-backed Reviewer with an Opus subagent dispatched via the Task tool. Removes the external `openai-codex` plugin dependency and the associated companion-runtime plumbing while preserving the "independent reviewer" guarantee (now via Sonnet → Opus tier jump + subagent context isolation, instead of cross-vendor model).

### Changed (Breaking)

- Reviewer now runs as an Opus subagent dispatched via the Task tool. Independence is preserved through a Sonnet → Opus tier jump plus subagent context isolation rather than a cross-vendor model.
- `code-feat` Step 6 collapses the previous 6a (Codex code quality) + 6b (Sonnet spec alignment + integration) split into a single Opus subagent that covers code quality, security, project conventions, spec alignment, and final output formatting in one pass.
- `code-review` `--change` mode's `grounding_rules` and `structured_output_contract` (previously embedded in the Codex `task` prompt) are absorbed into a unified Reviewer Subagent prompt template, shared across all scopes.
- `code-review` `--staged` mode no longer needs the manual fallback path that existed because Codex `review` did not support staged scope — all modes now go through the same subagent dispatch.
- Adversarial review is no longer a separate Codex `adversarial-review` subcommand; it is an `adversarial=true` flag on the same prompt template that switches on a red-team analysis section. Trigger conditions (security-sensitive paths, 2nd retry still failing, explicit user request, auto-escalation inside `code-feat`) are unchanged.
- WARNING re-check continues to run as a Sonnet subagent against `code-review`'s targeted-check mode (unchanged behavior).

### Removed

- `openai-codex` plugin dependency. README's prerequisite table no longer lists it.
- `codex-companion.mjs` path discovery (`~/.claude/plugins/cache/openai-codex/codex/*/scripts/`), `CODEX_COMPANION` env-var override, `--wait` background-job fallback (output-file polling + `status --json`), and the guidance against `result <jobId>` — all obsolete now that dispatch goes through the Task tool.
- Loading of `codex:codex-result-handling` and `codex:gpt-5-4-prompting` skills from the `code-review` flow.

### Migration

No state migration required. If `code@openai-codex` was installed solely to support `claude-sdd-kit`, it can be removed.

## 0.2.0 — 2026-05-04

Initial post-launch maintenance: fix command registration, align Coder skill rules between Tier 2 and Tier 3, and resync version metadata so `/plugin update` could pick up changes.

### Fixed

- Command files (`/code:feat`, `/code:fix`, `/code:review`) used non-standard frontmatter fields (`name`, `category`, `tags`) that Claude Code refused to register. Replaced with the standard `description` + `argument-hint` schema, and added `$ARGUMENTS` so commands forward arguments to their underlying skills.
- `marketplace.json` and `plugin.json` versions were out of sync, blocking `/plugin update` from detecting the new release. Both manifests now share the same version string.

### Changed

- Coder skill rules are now identical between `/code:feat` (Tier 3) and `/code:fix` (Tier 2): `vue-best-practices` moved from optional to required in `code-fix`, so cross-file fixes don't drift in style from feature work. Tier differences are now expressed in flow (Spectra integration, Reviewer) rather than in style strictness.
- `pnpm` / `turborepo` skill triggering now uses objective signals (`package.json`'s `packageManager` field, `pnpm-lock.yaml`, `turbo.json`) instead of subjective task-content heuristics.
- `web-design-guidelines` skill triggering narrowed to `.vue` `<template>` / `<style>` blocks or pure stylesheet files (`.css` / `.scss` / `.sass` / `.less`); pure `<script>` changes no longer trigger UI/a11y review skill loading.

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
