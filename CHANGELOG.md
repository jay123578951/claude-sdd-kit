# Changelog

## 0.7.1 — 2026-06-23

強化 `code-feat`、`code-fix` 的 skill description，讓「該用哪一個」的路由判斷隨 plugin 出貨、裝好即可靠運作，不依賴專案 CLAUDE.md 額外設定。Claude Code 安裝 plugin 後會自動以 skill 的 `name + description` 判斷適用情境——原描述只標 Tier 等級，缺鑑別條件與互相指路，AI 需靠語意猜測；本次補上判準，讓自動建議不靠運氣。

### Changed

- `code-feat` description 補上鑑別條件（5+ 檔或跨模組、需設計決策、新增 API/元件）、前提（需已有 OpenSpec change：`openspec/changes/<name>/` 含 `tasks.md`）與指路（小改動改用 `code-fix`）。
- `code-fix` description 補上鑑別條件（2-5 檔、無需設計決策、不新增 API/元件）與指路（需完整設計流程改用 `code-feat`；單行/純樣式微調直接在對話改）。
- `commands/feat.md`、`commands/fix.md` 的 frontmatter description 同步更新，與對應 skill 保持一致。

### Notes

- 此為路由可靠度的改善，不動 pipeline 行為（派發、retry、model 升級規則皆不變）。
- 專案特定慣例（UI 語言、設計系統、CSS 變數）仍由 runtime 從專案 CLAUDE.md 讀取——通用流程出貨於 kit、專案知識留在 CLAUDE.md 為刻意的分層。

## 0.7.0 — 2026-06-16

新增 `code-guidelines` 行為守則 skill，由 Coder 在動手前載入，針對 LLM 寫 code 的通病（亂假設、過度設計、亂改不該動的地方），從生成端先自我約束。原本這些性質只由 Reviewer 在事後攔截（過度抽象、只改必要），現在從生成端先自我約束——預防比 retry 便宜（少一輪 Opus review + 重跑測試）。skill 分類從「知識型 / 流程型」擴為加上「行為型」第三類。

### Added

- 新增 `plugins/code/skills/code-guidelines/SKILL.md`：定義 Coder 的四條行為守則——①自主判斷邊界（spec 沒寫死的技術細節自己決定、把假設寫進輸出摘要，只有觸及功能方向／缺人類資訊／不可逆操作才 STOP）、②最小可行實作、③外科手術式改動、④目標導向交付。

### Changed

- `code-feat`、`code-fix` 的 Coder 必載 skills 加入 `code-guidelines`（置於知識型 skill 之前，先讀再動手）；Step 4 / Step 2 的 Coder prompt 載入行同步更新，並在 Guardrails 補「含 retry 派發一律先載入行為守則」。
- `ai-development-pipeline.md`：Coder skill 載入表、Coder Agent 段同步加入 `code-guidelines`；「知識型 vs 流程型分離」段擴為「知識型 vs 流程型 vs 行為型」三類，並說明 `code-guidelines`（生成端自律）與 `code-review`（審查端把關）為同源配對。
- README 說明段與 Agent 編排段補述行為守則層；`plugin.json`、`marketplace.json` description 加註 `code-guidelines`。

### Notes

- 行為守則僅由 Coder 載入（唯一的 code 生成者）。Reviewer 已在審查端檢查同一組性質、Tester／註解整理各有專屬守則，故不重複載入以避免冗餘。
- 第①條「自主判斷邊界」刻意偏向讓 AI 自主完成：技術細節自己拍板、把假設寫進摘要而非停下問人，僅在功能方向／缺人類資訊／不可逆操作時才中斷。

## 0.6.0 — 2026-06-12

解除對 Spectra 的工具耦合，改為由 OpenSpec 變更 artifact 驅動。原本 skill 會直接呼叫 `spectra` CLI 並提示 `spectra:*` 指令，導致必須額外安裝 Spectra；實際上 pipeline 依賴的只是 `openspec/` 目錄與 artifact 格式（Spectra 底層就是 OpenSpec）。本次把前端工具綁定拿掉，讓 plugin 更單純、少裝一層。

### Changed

- `code-feat` Step 1/2 改為工具無關：不再執行 `spectra list --json` / `spectra status --json`，改為讀取 `openspec/changes/` 目錄與確認 `tasks.md` 存在；下一步提示由 `/spectra:verify`、`/spectra:archive` 改為 `/opsx:verify` → `/opsx:sync` → `/opsx:archive`。
- `code-fix`、`code-review` 的描述性 `Spectra artifacts` 字眼一律改為中性的「變更 artifact」或「OpenSpec 變更 artifact」。
- `ai-development-pipeline.md` 工具依賴表、Phase 1/4 流程、Skill 架構表改以 OpenSpec 指令（explore、propose、apply、verify、sync、archive）描述；Phase 4 因 OpenSpec 的 sync 與 archive 為獨立步驟，補回 `/opsx:sync`。
- README 前置依賴表 `Spectra CLI（必裝）` 改為 `OpenSpec（建議）`，最小範例與交付步驟改用 `/opsx:*` 指令。
- `plugin.json`、`marketplace.json` 的 description 從「with Spectra integration」改為「driven by OpenSpec change artifacts」。

### Removed

- 刪除文件中 OpenSpec 無對應的 Spectra 專屬步驟：Phase 1.5（analyze / clarify）與「跨階段 Spectra Skills」（debug / tdd），以及 `ingest` 提示。

### Notes

- 「工具無關」僅限 OpenSpec 相容的前端工具（OpenSpec CLI、Spectra、手刻）；skill 仍依賴 OpenSpec 的目錄與檔名約定（`openspec/changes/<name>/` + `proposal.md` / `design.md` / `tasks.md` / `specs/`），故 spec-kit 等不同 artifact 契約的工具無法直接相容。
- 「實驗記錄」與既有版本的 changelog 屬歷史敘述，保留當時的 `spectra` 用詞不動。

## 0.5.2 — 2026-06-08

`code-comment` 冗餘判準強化：把核心判準的評估視角明確定為「功能完成後、不知道開發過程的讀者」，並讓「冗餘」涵蓋語意複述而非僅字面直譯——解決「作用已由語意化 class／handler 名稱表達、卻仍寫一條標籤式註解」這類抓不掉的狀況。以強化單一大原則達成，不新增 case 規則。

### Changed

- `code-comment` 核心判準開頭改以「完成後讀者」視角評估：凡讀 code 本身（含語意化命名、結構，必要時對照鄰近檔案如 CSS／型別）即可在合理成本內看懂者皆為冗餘；唯跨越開發期仍成立的「為什麼」保留。判準從「寫的當下有沒有價值」改為「留在成品裡對未來讀者有沒有增益」；安全閥維持「拿不準就保留並列入保留決策覆核」。
- `冗餘複述` bad pattern 特徵從「字面直譯」放寬為「複述已由命名／結構表達的作用」，涵蓋語意複述（如 `// 品牌大圖覆蓋層` 對 class `logo-page-overlay`），未新增表格列。
- README 指令表與 Agent 編排段同步描述新判準。

## 0.5.1 — 2026-06-08

lint / test 指令一致化：把 `code-feat`、`code-fix` 既有的 Coder / Tester 指令從硬寫 `npx` 改為偵測專案 package manager 與 scripts，與 `code-comment` 收尾步驟的做法統一。

### Changed

- `code-feat`、`code-fix` 的 Coder prompt 不再寫死 `npx eslint --fix`，Tester prompt 不再寫死 `npx vitest run`；改為「優先專案 script（如 `pnpm lint --fix`、`pnpm test`）→ 無對應 script 才 fallback 到 `pnpm exec` → 避免裸 `npx`」。
- `ai-development-pipeline.md` 的「ESLint 三層防線」新增「指令執行慣例（lint / test 共用）」小節作為單一規範來源；Coder Agent 職責、Pipeline 圖、註解整理段同步改為「專案 lint script」。
- README Agent 編排段的註解整理安全網描述同步。
- 概念性提及 ESLint（git hook、CI、三層防線命名）維持不變——antfu 技術棧的 linter 本來就是 ESLint，本次只統一「怎麼呼叫」。

## 0.5.0 — 2026-06-08

新增 `code-comment` 註解整理 skill，並掛進 Tier 2 / Tier 3 Pipeline 的開發收尾。解決 AI 開發過程中不斷疊加註解、把思考流程寫進註解、註解過時沒跟上 code 的問題——收尾時以 fresh-eyes subagent 清除壞註解，只留說明「為什麼」與功能型指令的有效註解。

### Added

- 新增 `code-comment` skill（`plugins/code/skills/code-comment/SKILL.md`）與 `/code:comment` command。透過 Task tool 派發 **Sonnet 整理 Agent subagent**，與主對話隔離取得 fresh-eyes 視角（避免「剛寫完註解的人最難判斷哪些是廢話」的自評盲點）。
- 整理判準（single source of truth）：清除過時/矛盾、疊加殘留、思考流程口吻、冗餘複述、註解掉的死碼、空泛 TODO；保留解釋「為什麼」、JSDoc/docstring、複雜演算法說明、授權標頭、具體 TODO/FIXME。原則「刪除是危險方向，拿不準就保留」；修正過時註解時明令「不可改寫成只複述 code 的 what-comment」。
- 功能型指令註解保護清單涵蓋 linter/格式器（`eslint-disable`、`prettier-ignore`、`biome-ignore`、`stylelint-disable`）、TypeScript（`@ts-expect-error`/`@ts-ignore`/`@ts-nocheck`）、測試/coverage（`@vitest-environment`、`istanbul ignore`、`c8 ignore`）、bundler/編譯 pragma（`@__PURE__`、`@jsx`/`@jsxImportSource`、`@vite-ignore`、`webpackChunkName`、`@preserve`/`@license`）、框架（`v-html` 安全註記）——這些被誤刪一般測試不一定抓得到。
- 與 `code-review`（report-only + STOP）不同，整理 Agent 依守則**直接套用 Edit** 並自跑 lint --fix；orchestrator 在整理後**重跑測試**作為安全網（誤刪功能型指令註解會在此暴露）。安全網指令依專案 package manager / scripts 偵測（優先 `pnpm lint` / `pnpm test`），不寫死 `npx`。
- 掃描範圍分級：獨立模式預設只清 diff 改動區及其鄰近註解（避免誤傷他人 ownership 的舊碼），`--whole-file` 才放寬；Pipeline 模式因檔案是該 pipeline 自己寫的，允許整個 changed file。

### Changed

- `code-feat`（Tier 3）新增 Step 6.5「註解整理」：Reviewer PASS 後派發 Sonnet 整理 Agent，retry 全部 settle 後一次清最終狀態。Model 策略表、Phase 2 完成輸出、Guardrails 同步更新。
- `code-fix`（Tier 2）新增 Step 4.5「註解整理」：Coder/Tester settle 後、Spec 影響檢查前執行。Model 策略表、完成輸出、Guardrails 同步更新。
- `ai-development-pipeline.md` 新增「註解整理 Agent」章節，更新 Phase 2 Pipeline 圖、Model 分層策略表、Agent Knowledge Skills 載入表、Tier 2/3 流程圖。
- README 指令表與 Agent 編排段補上 `/code:comment`。

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
