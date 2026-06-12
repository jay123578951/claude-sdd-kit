# claude-sdd-kit

> 為 Vue/Nuxt 專案打造的 Claude Code plugin — 用一個指令跑完 Coder → Tester → Reviewer 的 SDD 工作流。

## 這是什麼

`claude-sdd-kit` 把 Spec-Driven Development 的編排成本壓成一個指令。搭配 [OpenSpec](https://github.com/Fission-AI/OpenSpec) 管理變更規格，Reviewer 透過 Opus subagent 進行獨立 code review。

手動跑 SDD 流程時要自己載 skills、切換 Coder / Tester / Reviewer 三種角色、把 design 與 tasks 貼進對話、最後還要確保 spec 跟 code 同步——這些都能編排，但每次手動拼裝太累。更重要的是 **reviewer 應該獨立**：Coder / Tester / Reviewer 各自派發成獨立 subagent，Reviewer 固定走 Opus 並與其他角色 context 隔離，靠獨立視角避免自評自審讓品質失真。Coder 則依變更性質**動態選模型**——一般功能用 Sonnet，碰到架構變更、安全敏感路徑、設計決策密集或 retry 第 2 輪才升 Opus。所以這個 plugin 把 agent 派發、skill 載入、獨立 review 全包起來，並用**變更分級制度**避免小改動被完整 spec 流程綁住——改個 CSS 就用 `/code:fix`，做新功能才走 `/code:feat`。

## Features

### 指令

| 指令 | 適用情境 | 流程 |
|------|----------|------|
| `/code:feat <change-name>` | 新功能、大型重構、跨模組變更 | Coder → Tester → Reviewer → 註解整理，搭配 OpenSpec 變更 artifact |
| `/code:fix` | 跨檔案 bug fix、小型 UI 調整、composable 微調 | Coder + Tester → 註解整理，含 Spec 影響檢查 |
| `/code:review [--staged \| --branch <ref> \| --change <name>]` | 獨立 code review | Opus subagent 獨立審查（與主對話 context 隔離） |
| `/code:comment [--staged \| --branch <ref> \| --whole-file]` | 開發收尾清理註解 | Sonnet subagent 以「完成後讀者」視角清除過時/冗餘/思考流程註解（冗餘含語意複述，非僅字面直譯），保留 why 與功能型指令（獨立模式預設只清 diff 鄰近，`--whole-file` 放寬到整檔） |

> 微調（CSS、文字、單行 fix）建議直接在主對話改，不需走 plugin。完整分級判斷見 [`plugins/code/docs/ai-development-pipeline.md`](plugins/code/docs/ai-development-pipeline.md)。

### Agent 編排

- **Coder**（Sonnet → Opus 動態切換）— 預設 Sonnet subagent；首次派發前若判定為架構變更 / 安全敏感路徑 / 設計決策密集則升 Opus，retry 進入第 2 輪起也自動升 Opus（後續不再降回）。`/code:feat` 與 `/code:fix` 共用同一套 skill 規範：必載 `vue` / `vue-best-practices` / `nuxt` / `antfu`，依任務追加 `pinia` / `unocss` / `vite` / `vue-router-best-practices` / `vueuse-functions` / `pnpm` / `turborepo`。Tier 差異在流程而非風格寬鬆度
- **Tester**（Sonnet）— 自動載入 `vitest` / `vue-testing-best-practices`，測試失敗會退回 Coder 修復（最多 3 輪）
- **Reviewer**（Opus subagent）— Task tool 派發 Opus subagent 一次審完 code quality / 安全性 / 慣例 / spec alignment；改動觸及 `.vue` template/style 或純樣式檔時加載 `web-design-guidelines` 補 UI/a11y 檢查；安全敏感路徑或第 2 輪 retry 仍 FAIL 時自動升級為 adversarial prompt；FAIL 時退回對應 agent，WARNING re-check 降級為 Sonnet targeted check
- **註解整理**（Sonnet subagent）— 開發收尾 fresh-eyes 清除 AI 累積的過時/疊加/思考流程/冗餘註解，以「功能完成後、不知道開發過程的讀者」視角評估：凡讀命名／結構／鄰近檔案（如 CSS／型別）即可回推者皆視為冗餘（涵蓋語意複述，不限字面直譯），唯跨越開發期仍成立的「為什麼」、JSDoc 與功能型指令註解（`eslint-disable`、`@ts-expect-error` 等）保留；依守則直接套用 Edit 後重跑 lint + 測試作安全網（指令依專案 package manager / scripts 偵測，不寫死 `npx`）

### Spec 同步保證

- `/code:feat` 自動更新 `openspec/changes/<name>/` 下的 design 與 tasks
- `/code:fix` 在 commit 前比對 `openspec/specs/` 與 `docs/plans/`，確保 code 與 spec 不脫鉤

## Install

### 前置依賴

| 工具 | 用途 |
|------|------|
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | 產出/管理 `openspec/` 變更 artifact（建議；`/code:feat` 需要變更目錄已存在） |
| [antfu/skills](https://github.com/antfu/skills) | 提供 `vue` / `nuxt` / `antfu` / `vitest` / `vue-testing-best-practices`（必裝） |

### 安裝本 plugin

```bash
/plugin marketplace add jay123578951/claude-sdd-kit
/plugin install code@claude-sdd-kit
```

### 驗證

```bash
/plugin list                       # 應看到 code@claude-sdd-kit
ls ~/.claude/skills                # 應有 vue / nuxt / antfu / vitest / vue-testing-best-practices
```

## 最小範例

```bash
/opsx:explore weather-monitor       # 探索需求（可選）
/opsx:propose weather-monitor       # 產出 proposal / design / tasks / specs

/code:feat weather-monitor          # Coder → Tester → Reviewer 一氣呵成
```

跑完後人工驗收，最後 `/opsx:verify` → `/opsx:sync` → `/opsx:archive` → commit → merge。

## 專案慣例

UI 語言、設計系統、CSS 變數命名等專案特有慣例請寫在根目錄 `CLAUDE.md`，agent 派發前會自動讀取，並在 review 階段再次引用。

## Feedback

Bug 或建議請開 [GitHub Issues](https://github.com/jay123578951/claude-sdd-kit/issues)。

## License

MIT — 見 [LICENSE](LICENSE)。變更紀錄見 [CHANGELOG.md](CHANGELOG.md)。
