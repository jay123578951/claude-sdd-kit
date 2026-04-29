---
name: code-review
description: Use when reviewing code changes for quality, security, and project conventions — standalone or as review standard within code-feat pipeline
---

獨立的 Code Review skill，定義 review 維度、流程與輸出格式。可在任何場景獨立呼叫，也作為 `code-feat` Reviewer Agent 的 review 標準單一來源。

**Input**: 可選指定 review 範圍（見下方模式）。未指定時自動偵測 git diff。

**依賴**：本 skill 透過 [openai-codex plugin](https://github.com/openai/codex) 執行 Codex review。請先安裝該 plugin（見本 plugin README 的安裝說明）。

**成本提示**：標準流程透過 Codex 執行 review，每次呼叫會消耗 Codex quota。以下情境建議略過此 skill，直接請 Claude 本地讀檔給意見：

- 改動範圍僅 1–2 個小檔案（成本划不來）
- 單純格式/文字修正（ESLint/TypeScript 已能捕捉）
- 想要快速 sanity check 而非正式 review

---

## Review 模式

根據輸入自動判斷：

```
/code:review                     → 自動偵測：git diff 未 commit 的變更
/code:review --staged            → 只看 staged changes
/code:review --branch feat/xxx   → 整個 branch 相對 main 的 diff
/code:review --change xxx        → 讀取 Spectra change artifacts 作為 review 基準
```

被 `code-feat` 的 Reviewer Agent 載入時，由 Reviewer prompt 指定範圍，不需自動偵測。

---

## 流程

### Step 1: 準備

1. 讀取專案的 CLAUDE.md 了解專案慣例
2. 依 review 模式確認變更範圍：
   - 自動偵測：`git diff` + `git diff --staged`
   - `--staged`：`git diff --staged`
   - `--branch`：`git diff main...<branch>`
   - `--change`：讀取 `openspec/changes/<name>/` 下的所有 artifacts

### Step 2: 載入 Codex skills

依模式條件載入：

- **所有模式**：`codex:codex-result-handling`（負責呈現 Codex 輸出與 STOP 規則）
- **`--change` 模式追加**：`codex:gpt-5-4-prompting`（需要自組 `task` prompt 時才需要；標準 `review` / `adversarial-review` 命令已內建 review contract，不需要此 skill）

### Step 3: 執行 Review（透過 Codex）

由於本 skill 屬於另一個 plugin，無法直接使用 `${CLAUDE_PLUGIN_ROOT}` 解析到 openai-codex plugin 的腳本路徑。請動態定位：

```bash
# 自動探索最新版 openai-codex plugin 的腳本位置（可改用 CODEX_COMPANION 環境變數覆寫）
CODEX="${CODEX_COMPANION:-$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)}"
[ -z "$CODEX" ] && { echo "ERROR: openai-codex plugin 未安裝。請執行 /plugin install codex@openai-codex" >&2; exit 1; }
```

依 review 模式選對應指令：

| 模式 | 指令 |
|------|------|
| 自動偵測 | `node "$CODEX" review --wait --scope auto` |
| `--branch feat/xxx` | `node "$CODEX" review --wait --scope branch --base main` |
| `--staged` | **Codex `review` 不支援 staged scope**。由主 agent 讀 `git diff --staged` 並依下方 Review 維度手動執行，不走 Codex |
| `--change xxx` | 走 `task` 命令，見下方「`--change` 模式」小節 |
| 深度對抗 | `node "$CODEX" adversarial-review --wait --scope auto`（或加 `--scope branch --base main`）|

所有 Codex 呼叫一律使用 `--wait`。若 `--wait` 正常 block，stdout 即為完整輸出，直接進 Step 4。

**`--wait` 未 block 時的 fallback（stdout 含 "running in background"）**：

1. 從 stdout 擷取 output 檔案路徑（`"Output is being written to: <path>"`）
2. 輪詢 `node "$CODEX" status --json`，等待該 job 的 `phase` 離開 `"running"`（間隔 5 秒，最多 5 分鐘）
3. 讀取 output 檔案路徑（`cat <path>`），完整內容即為 Codex review 輸出
4. **不要使用 `result <jobId>`**——已知在部分環境下無法找到 job

被 `code-feat` Reviewer Agent 載入時，scope/base 由呼叫方 prompt 指定。

**adversarial-review 觸發條件**（滿足任一即升級）：

- 安全敏感路徑（auth、payment、API key 處理、session 管理）
- 資料庫 schema 變更或生產資料遷移
- 使用者明確要求深度 review
- `code-feat` Reviewer 在第 2 輪 retry 仍 FAIL 時升級

**Codex 執行失敗 / 無法啟動時**：

- 獨立模式：依 `codex:codex-result-handling` 規則，報告失敗並停止。不要改由 Claude 自行做一次替代 review
- `code-feat` 模式：Reviewer Agent 收到空 Codex 輸出繼續執行 Spec alignment 與專案慣例檢查（由呼叫方 pipeline 決定）

#### `--change` 模式

需把 Spectra artifacts 塞進 `task` prompt 作為 review 基準。呼叫方代入 `{changeName}` 後執行：

```bash
node "$CODEX" task --wait --effort high "$(cat <<'PROMPT'
<task>
對 Spectra change `{changeName}` 進行 Root-Cause Review。驗證當前 git working tree 的實作是否符合 proposal/design/specs 的規劃。

變更目錄：openspec/changes/{changeName}/

先讀取以下 artifacts：
- proposal.md
- design.md
- tasks.md
- specs/ 下的 delta spec 檔案

再對照 `git diff` 的變更內容進行 review。
</task>

<grounding_rules>
- 每個 finding 必須指到明確的檔案:行號
- requirement / scenario 未實作時，必須引用 spec 原文作為證據
- 不要推測「可能的問題」。只報告有證據支持的事實
- 區分 observed fact 與 inference，inference 必須明確標註
</grounding_rules>

<structured_output_contract>
輸出三段：

1. Spec Alignment 檢核表：每個 requirement 的狀態（已實作 / 部分實作 / 未實作），未實作者附 spec 原文引用

2. 問題清單表格，欄位固定為：
   | # | 嚴重度 | 歸屬 | 檔案:行號 | 描述 | 修復建議 |
   - 嚴重度：CRITICAL / WARNING / SUGGESTION
   - 歸屬：coder（實作代碼）/ tester（測試代碼）
   - 按嚴重度排序

3. 1-2 句整體摘要
</structured_output_contract>

<default_follow_through_policy>
不要詢問澄清問題。若資訊不足，直接在輸出中標記為 open question，繼續完成其餘檢查。
</default_follow_through_policy>
PROMPT
)"
```

prompt 中的 review 維度由主 agent 依下方「Review 維度」章節補入 `<task>` 段落（或依需要補 `<grounding_rules>`）。主 agent 收到 Codex 輸出後，依本 skill「輸出格式」章節重新包裝為最終報告。

### Step 4: 呈現結果

依 `codex:codex-result-handling` 呈現 Codex 輸出，並對應到下方定義的嚴重程度與輸出格式。

**獨立模式下的後處理責任**（由主 agent 執行，非 Codex）：

1. **重新包裝輸出**：將 Codex 原始 findings 套用到下方「輸出格式」章節的表格結構，下最終 PASS / PASS with WARNING / FAIL 判定
2. **補充專案慣例檢查**：Codex 的通用 review 容易遺漏專案特有檢查項目，主 agent 必須對照「Review 維度 → 必檢 → 專案慣例」欄目，逐項檢查 Codex 是否已涵蓋：
   - 繁體中文 UI 文字（Codex 預設英文環境，容易漏判）
   - CSS 變數使用（不硬編碼顏色、間距、字體大小）
   - 遵循專案設計系統慣例（class 命名、token、元件結構）
   - 專案 CLAUDE.md 中定義的其他特定慣例
3. **合併輸出**：Codex findings + 主 agent 補充 findings 一起呈現，並在「摘要」段註明哪些由補檢發現

被 `code-feat` Reviewer Agent 載入時，此後處理由 Reviewer Agent (Sonnet) 負責，主 agent 不需執行。

**STOP 規則（來自 codex-result-handling）**：呈現 findings 後立即停止。不修復任何問題。必須明確詢問使用者要修什麼才能動檔案。

---

## Review 維度（供 `--change` 模式 task 提示使用）

### 必檢（所有模式）

| 維度 | 檢查項目 |
|------|---------|
| **程式碼品質** | 命名一致性、函式/元件結構、可讀性、重複邏輯、過度抽象 |
| **安全性** | API key 暴露、XSS、注入風險、敏感資料洩漏 |
| **專案慣例** | 繁體中文 UI 文字、CSS 變數使用（不硬編碼顏色）、遵循專案設計系統慣例、專案 CLAUDE.md 中的其他規則 |

### 條件檢查

| 維度 | 觸發條件 | 檢查項目 |
|------|---------|---------|
| **測試品質** | 變更含測試檔案 | 測試是否有效驗證行為（非複製邏輯自測）、排除規則是否遵守 |
| **Spec 一致性** | 有 Spectra change（`--change` 模式或由 `code-feat` 指定） | requirements 是否全部實作、scenarios 是否覆蓋、design 決策是否遵循 |

### 不檢查

- TypeScript 型別正確性（編譯器負責）
- ESLint 能捕捉的格式問題（lint 負責）
- 效能最佳化（除非有明顯的 O(n²) 或記憶體洩漏）

---

## 嚴重程度定義

| 等級 | 定義 | 處理方式 |
|------|------|---------|
| **CRITICAL** | 會導致功能錯誤、安全漏洞、或資料遺失 | 必須修復 |
| **WARNING** | 違反專案慣例、影響可維護性、或潛在風險 | 需要修復 |
| **SUGGESTION** | 可改善但不影響正確性的建議 | 僅記錄，不要求修復 |

---

## 輸出格式

```
## Code Review：{scope 描述}

### 判定：PASS / PASS with WARNING / FAIL

### 問題清單

（若有 CRITICAL 或 WARNING）

| # | 嚴重度 | 歸屬 | 檔案:行號 | 描述 | 修復建議 |
|---|--------|------|----------|------|---------|
| 1 | CRITICAL | coder | foo.vue:42 | ... | ... |
| 2 | WARNING | tester | foo.test.ts:10 | ... | ... |

### SUGGESTION

（僅記錄，不要求修復）

- foo.vue:15 — ...
- bar.ts:30 — ...

### 摘要

{1-2 句整體評價}
```

### 歸屬欄位

- `coder` — 實作代碼問題
- `tester` — 測試代碼問題

歸屬用於 `code-feat` pipeline 的 retry 迴路，決定派發給哪個 agent 修復。獨立使用時僅作為參考。

### 判定規則

- 有任何 CRITICAL → **FAIL**
- 有 WARNING 但無 CRITICAL → **PASS with WARNING**
- 只有 SUGGESTION 或無問題 → **PASS**

---

## Targeted Check 模式

當被用於 `code-feat` 的 WARNING re-check 時，執行精簡版 review：

- **由 Sonnet agent 直接讀檔執行，不呼叫 Codex**（避免為小範圍 re-check 消耗 Codex quota，並避開 Codex 背景作業的延遲）
- 只讀取改動的檔案和對應行數
- 只驗證原始 WARNING 是否已正確修復
- 不重新掃描所有檔案
- 輸出格式相同，但 scope 描述標記為「targeted re-check」

---

## 與 Pipeline 的關係

| 使用者 | 如何使用 |
|--------|---------|
| `code-feat` Step 6 | Reviewer Agent 載入此 skill，執行完整 review |
| `code-feat` WARNING re-check | Sonnet agent 載入此 skill，執行 targeted check |
| `code-fix`（可選） | 完成後人工決定是否跑 `/code:review` |
| 獨立使用 | 任何時候對任意 diff 執行 review |

此 skill 只負責「怎麼 review」。派發邏輯（retry、model 選擇、3 輪上限）由呼叫方（`code-feat` 或人）管理。
