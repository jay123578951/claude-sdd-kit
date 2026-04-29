# claude-sdd-kit

A Claude Code plugin marketplace for **Spec-Driven Development (SDD)** pipelines, designed around Vue/Nuxt projects with Spectra integration.

## Overview

This marketplace provides plugins that implement a tiered AI development pipeline. The included `code` plugin packages three workflow skills used to drive coding work through Claude Code's agent system:

| Skill | Tier | Purpose |
|-------|------|---------|
| `code-feat` | Tier 3 | Full pipeline (Coder → Tester → Reviewer) for feature work driven by Spectra change artifacts |
| `code-fix` | Tier 2 | Lightweight pipeline (Coder + Tester) for cross-file bug fixes and small adjustments |
| `code-review` | — | Standalone review skill, also used as the review standard within `code-feat` |

The full methodology is documented in [`plugins/code/docs/ai-development-pipeline.md`](plugins/code/docs/ai-development-pipeline.md).

## Installation

### Prerequisites

This plugin depends on the following plugins. Install them first:

```bash
# Codex integration (used by code-feat / code-review for code review)
/plugin marketplace add openai/codex
/plugin install codex@openai-codex
```

> The `code-feat` and `code-review` skills invoke `codex-companion.mjs` from the `openai-codex` plugin to run Codex code reviews. The skills auto-discover the latest installed version under `~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs`. Override with the `CODEX_COMPANION` env var if needed.

It also expects the [Spectra CLI](https://github.com/Fission-AI/Spectra) to be installed (the skills call `spectra` commands directly).

### Install this plugin

```bash
/plugin marketplace add jay123578951/claude-sdd-kit
/plugin install code@claude-sdd-kit
```

After install, the following slash commands become available:

- `/code:feat [change-name]` — run the Tier 3 pipeline against a Spectra change
- `/code:fix` — run the Tier 2 pipeline against a problem described in conversation
- `/code:review [--staged|--branch <ref>|--change <name>]` — run a standalone review

## Project Conventions

The skills assume Vue/Nuxt projects and load the following knowledge skills (from [antfu/skills](https://github.com/antfu/skills)) when dispatching agents:

- Coder: `vue`, `nuxt`, `antfu` (plus `pinia` / `unocss` / `vite` / `vue-router-best-practices` when the task warrants)
- Tester: `vitest`, `antfu`, `vue-testing-best-practices`

Project-specific conventions (UI language, design system rules, etc.) should live in each project's `CLAUDE.md`. The skills read `CLAUDE.md` before dispatching agents and reference it again during the project-conventions check in `code-review`.

## Plugin Structure

```
claude-sdd-kit/
├── .claude-plugin/
│   └── marketplace.json        # marketplace metadata
└── plugins/
    └── code/                   # the "code" plugin
        ├── .claude-plugin/
        │   └── plugin.json
        ├── commands/           # /code:feat, /code:fix, /code:review
        ├── skills/             # code-feat, code-fix, code-review
        └── docs/
            └── ai-development-pipeline.md
```

## Versioning

Semver. Breaking changes bump major; new skills bump minor; doc/text fixes bump patch.

## License

MIT — see [LICENSE](LICENSE).
