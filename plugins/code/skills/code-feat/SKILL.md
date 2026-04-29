---
name: code-feat
description: Tier 3 完整 Pipeline — 透過 Agent Pipeline 自動實作 Spectra 變更（Coder → Tester → Reviewer）
---

Tier 3 完整版 Agent Pipeline，適用於需要設計決策的完整功能（新功能、大型重構、架構變更）。透過 Spectra artifacts 驅動，派發三個專職 agent 分工執行。

小改動請改用 `/code:fix`（Tier 2）。

**Input**: 可選指定變更名稱（e.g., `/code:feat add-auth`）。未指定時從對話推斷或提示選擇。

**依賴**：本 skill 的 Step 6a 透過 [openai-codex plugin](https://github.com/openai/codex) 執行 Codex code review。請先安裝該 plugin（見本 plugin README 的安裝說明）。

---

## 專案配置

### Agent Knowledge Skills

| Agent | Skills（必載） | 可選 Skills | 用途 |
|-------|---------------|------------|------|
| Coder | `vue`, `nuxt`, `antfu` | 由 orchestrator 根據 task 內容預判並寫入 prompt（如 `pinia`, `unocss`, `vite`, `vue-router-best-practices`） | Vue/Nuxt 開發慣例、程式碼風格 |
| Tester | `vitest`, `antfu`, `vue-testing-best-practices` | — | 測試框架用法、Vue 元件測試慣例 |
| Reviewer | `code-review` | — | Reviewer Sonnet subagent 使用；Codex 負責 code quality，Sonnet 負責 spec alignment + 整合輸出 |

### Model 策略

| Agent | Model | 說明 |
|-------|-------|------|
| Coder | sonnet | |
| Tester | sonnet | |
| Reviewer 6a | Codex | Code quality、安全性、慣例（走 Codex quota） |
| Reviewer 6b | sonnet | Spec alignment check + 整合 Codex findings，輸出 PASS/FAIL/WARNING |

---

## 流程

### Step 1: 選擇變更

與 `spectra:apply` 相同：

1. 若有提供名稱，直接使用
2. 否則從對話推斷，或 `spectra list --json` 讓使用者選擇
3. 宣告：「Using change: <name>」

### Step 2: 確認狀態

```bash
spectra status --change "<name>" --json
```

- `isComplete: false` 且有未完成的前置 artifacts → 提示先用 `spectra:propose` 或 `spectra:ingest`
- tasks artifact 必須存在才能繼續

### Step 2.5: 建立工作分支

確認當前 git 分支狀態：

- 若在 `main`、`develop` 等主要分支上 → `git checkout -b feat/{changeName}`
- 若已在功能分支上 → 跳過

### Step 3: 評估任務規模與分批策略

讀取 tasks.md，以 **Task 大項（T1、T2、T3…）** 為單位評估：

- **單一大項**：整批派發給一個 Coder Agent
- **多個大項**：每個大項為一批，獨立走 Coder → Tester 流程，全部完成後再跑一次完整 Reviewer
- **大項之間有依賴關係時**（如 T2 使用 T1 產出的模組），按依賴順序串行執行
- **大項之間無依賴時**，可平行派發

依賴判斷優先從 task 描述推斷。**若無法明確判斷，預設為串行執行（保守策略）。** 只有在 orchestrator 確信無依賴時才平行派發。

Task 大項本身對應 spec 的模組邊界，天然適合作為分批單位。不以 checkbox 數量切分，避免把同一模組的邏輯拆散到不同 agent。

**跨批 Context 注意事項**

串行執行時，後續批次的 Coder prompt 須額外包含：
- 前批產出的檔案清單
- 前批 Coder 的關鍵設計決策摘要（來自 Step 4 的輸出）

目的：確保後批 agent 沿用前批建立的介面與慣例，而非僅靠讀取原始碼推斷。

Retry 修復時，若修改涉及跨批共用的介面（如 composable 的回傳結構），orchestrator 應重跑受影響批次的測試。

**Coder Skills 預判**

Orchestrator 根據 tasks.md 內容判斷 Coder 除了必載的 `vue`, `nuxt`, `antfu` 外，還需要哪些額外 skills，寫入 `{additionalSkills}` 變數。例如：
- task 涉及 store / state → 加 `pinia`
- task 涉及 CSS utility / atomic → 加 `unocss`
- task 涉及 build config / plugin → 加 `vite`
- task 涉及路由 / middleware / navigation guard / route params → 加 `vue-router-best-practices`

### Step 4: 派發 Coder Agent

使用 Task tool 派發 subagent（model: sonnet）：

```
你是 Coder Agent。

變更名稱：{changeName}
變更目錄：openspec/changes/{changeName}/
實作範圍：{taskList}（若為分批模式，僅列本批 task）

開始工作前：
1. 用 Skill tool 載入以下 skills：vue, nuxt, antfu{additionalSkills}
2. 讀取專案的 CLAUDE.md 了解專案慣例
3. 讀取變更目錄下的 design.md、tasks.md 和 specs/ 下的 delta spec 檔案

注意：若實作過程中發現需要用到上方未列出的 skill，可自行載入補充。

依照指定的 task 逐項實作：
- 遵循專案設計系統與慣例
- 善用既有的 composables 和 utils
- 每完成一個 task，更新 tasks.md 的 checkbox：`- [ ]` → `- [x]`

完成後執行 ESLint --fix 確保代碼通過 lint。

輸出：
1. 列出你建立/修改/刪除的所有檔案路徑
2. 簡述每個 task 的關鍵設計決策（供 retry 時參考）
```

### Step 5: 派發 Tester Agent

使用 Task tool 派發 subagent（model: sonnet）：

```
你是 Tester Agent。

變更名稱：{changeName}
變更目錄：openspec/changes/{changeName}/

Coder 產出的檔案：
{coderOutputFiles}

開始工作前：
1. 用 Skill tool 載入以下 skills：vitest, antfu, vue-testing-best-practices
2. 讀取變更目錄下的 specs/ 目錄（了解預期行為的 scenarios）
3. 讀取變更目錄下的 design.md（了解設計意圖，使測試貼近實作決策而非僅驗表面行為）
4. 讀取上方列出的 Coder 產出/修改檔案

撰寫測試：
- 測試檔放在與源碼同目錄（foo.test.ts）
- 使用 describe/it API 結構
- 針對 spec 中的每個 scenario 撰寫測試
- 純邏輯函式應抽出為獨立模組，測試 import 實際模組（不複製邏輯）

排除規則（不要撰寫以下測試）：
- TypeScript 型別/介面欄位存在性測試（TypeScript 編譯器已保證型別正確性）
- 無法 import 實際模組時（如 Nuxt composable），跳過該模組的單元測試，不要複製邏輯自測

完成後執行測試：npx vitest run --reporter=verbose

輸出：
1. 建立的測試檔案路徑
2. 測試結果（通過/失敗）
3. 若有失敗，列出每個失敗的測試名稱和錯誤原因
4. 無法測試的模組清單（列出模組名稱與跳過原因，如「useWeatherApi — Nuxt composable，無法在單元測試中 import」）
```

**測試失敗時**：進入 Retry 迴路（見下方）。

### Step 6: Reviewer（Codex + Sonnet）

#### Step 6a：Codex Code Review

執行 Codex review（前景等待，顯式 scope）。

由於本 skill 屬於另一個 plugin，無法直接使用 `${CLAUDE_PLUGIN_ROOT}` 解析到 openai-codex plugin 的腳本路徑。請動態定位：

```bash
# 自動探索最新版 openai-codex plugin 的腳本位置（可改用 CODEX_COMPANION 環境變數覆寫）
CODEX="${CODEX_COMPANION:-$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)}"
[ -z "$CODEX" ] && { echo "ERROR: openai-codex plugin 未安裝。請執行 /plugin install codex@openai-codex" >&2; exit 1; }
node "$CODEX" review --wait --scope auto
```

- `--wait`：嘗試同步等待；若正常 block，stdout 即為完整輸出
- `--scope auto`：code-feat 進到 Step 6 時 Coder/Tester 的改動尚未 commit，auto 會涵蓋 working-tree

**`--wait` 未 block 時的 fallback（stdout 含 "running in background"）**：

1. 從 stdout 擷取 output 檔案路徑（`"Output is being written to: <path>"`）
2. 輪詢 `node "$CODEX" status --json`，等待 job `phase` 離開 `"running"`（間隔 5 秒，最多 5 分鐘）
3. 讀取 output 檔案（`cat <path>`）取得完整 review 輸出
4. **不要使用 `result <jobId>`**——在部分環境下無法找到 job

若需升級深度對抗 review（例如第 2 輪 retry 仍 FAIL、或涉及安全敏感路徑），改用 `adversarial-review` 子命令，觸發條件見 `code-review` skill 的定義。

將完整輸出存為 `{codexReviewOutput}`。

若 Codex 執行失敗，記錄錯誤訊息並以空輸出繼續 Step 6b（不中止 pipeline）。Reviewer Agent 仍會執行 Spec alignment 與專案慣例檢查作為保底。

#### Step 6b：Spec Alignment + 整合輸出

使用 Task tool 派發 subagent（model: sonnet）：

```
你是 Reviewer Agent。

變更名稱：{changeName}
變更目錄：openspec/changes/{changeName}/

--- Coder 產出摘要 ---
修改的檔案：
{coderOutputFiles}

設計決策：
{coderDesignDecisions}

--- Tester 產出摘要 ---
測試檔案：
{testerOutputFiles}

測試結果：
{testerResults}

--- Codex Review 輸出 ---
{codexReviewOutput}

---

開始工作前：
1. 用 Skill tool 載入 `code-review` skill — 這是你的 review 標準與輸出格式的單一來源
2. 讀取變更目錄下的所有 artifacts（proposal.md、design.md、tasks.md、specs/）
3. 讀取上方列出的所有 Coder 產出檔案
4. 讀取專案的 CLAUDE.md 了解專案慣例

你的工作分兩部分：

**Part 1：Spec Alignment Check**
對照 Spectra change artifacts 驗證：
- 所有 requirements 是否已實作
- 所有 scenarios 是否有對應實作或測試覆蓋
- 設計決策是否遵循 design.md 的規劃

**Part 2：整合 Codex Findings**
整合 Codex review 的發現：
- 將 Codex 找到的問題對應到 code-review skill 的嚴重程度定義（CRITICAL / WARNING / SUGGESTION）
- 補充 Codex 可能遺漏的專案特有慣例（繁體中文 UI 文字、CSS 變數使用、專案設計系統慣例、專案 CLAUDE.md 中定義的規則）

依照 code-review skill 定義的輸出格式產出最終 review 報告。
```

**Review 不通過時**：進入 Retry 迴路（見下方）。

### Step 7: 報告結果

顯示 Phase 2 完成摘要，提示進入 Phase 3 人工驗收。

---

## Retry 迴路

### 測試失敗 → Coder 修復

1. 將以下資訊傳給 Coder Agent：
   - 失敗的測試名稱和錯誤訊息
   - **前一輪 Coder 的輸出摘要**（修改的檔案清單 + 關鍵設計決策）
   - 明確的修復指示
2. Coder prompt 須明確指示：**不需重讀 design.md / specs/**，只讀前輪輸出摘要、失敗報告、要修改的檔案（節省重複 context 載入）
3. Coder 修復後，重新派發 Tester Agent 驗證（Tester 亦不需重讀 design.md / specs/，僅讀 Coder 本輪修改的檔）
4. 最多 3 輪，修不好 → 停下來問人

### Review FAIL（有 CRITICAL）

依問題歸屬派發修復：
- 程式碼品質問題 → Coder Agent 修復
- 測試品質問題 → Tester Agent 修復
- 安全性問題（嚴重）→ 停下來問人

修復 agent 的 prompt 需包含前一輪的輸出摘要（檔案清單 + 設計決策），避免 context 斷裂。**修復 agent 不需重讀 design.md / specs/**，只讀前輪輸出摘要、Review 報告、要修改的檔案。

修復後重新派發 Reviewer Agent 驗證。最多 3 輪。

### Review PASS with WARNING

- WARNING 視為需修復
- **同一歸屬的所有 WARNING 合併為一個修復任務**（Coder 一次收到所有歸屬它的 WARNING 清單，一次改完）
- 問題歸屬：實作代碼 → Coder，測試代碼 → Tester
- Coder 和 Tester 的修復任務若同時存在，可平行派發
- 修復後的 re-check 使用 **Sonnet model**（非完整 Opus Reviewer）
- re-check agent 同樣載入 `code-review` skill，但執行 targeted check 模式：只讀取改動的檔案和行數，驗證修復是否正確，不重新掃描所有 artifacts

### Retry 計數

- ESLint 錯誤：Coder 自行修復，不計入重試次數
- 測試失敗：Coder ↔ Tester 最多 3 輪
- Review 不通過：最多 3 輪
- 3 輪修不好 → 停下來問人（問題可能較嚴重或 AI 忽略關鍵細節）

---

## 輸出格式

### Phase 2 完成

```
## Phase 2 完成：{changeName}

### Agent Pipeline 結果
- Coder: ✓ 完成（N 個檔案）
- Tester: ✓ 通過（M 個測試）
- Reviewer: ✓ PASS

### Pipeline 統計
- 分批：{batchCount} 批（或「單批」）
- Coder 派發次數：{coderCalls}（含 retry）
- Tester 派發次數：{testerCalls}（含 retry）
- Reviewer 派發次數：{reviewerCalls}（含 retry）

### Retry 記錄
（若有 retry，列出每輪的問題與修復摘要）

### 下一步
進入 Phase 3 人工驗收。請啟動 dev server 測試功能。
驗收通過後，執行：
1. /spectra:verify
2. /spectra:archive
```

### 遇到阻塞

Orchestrator 在宣告阻塞前，輸出 debug 檔案 `debug-{changeName}-{timestamp}.md`（放在專案根目錄）：

```markdown
## Pipeline 暫停：{changeName}

### 阻塞原因
{最後一輪的錯誤訊息}

### Retry 歷程

#### Round 1
- Coder 修改：{檔案清單}
- 結果：{失敗原因}

#### Round 2
...

#### Round 3
...

### 當前 Diff
（附上 git diff 輸出）
```

然後向使用者顯示阻塞摘要：

```
## Pipeline 暫停：{changeName}

### 問題
{問題描述}

### 已完成
- Coder: ✓
- Tester: ✗ 第 3 輪仍失敗

### Pipeline 統計
- Coder 派發次數：{coderCalls}
- Tester 派發次數：{testerCalls}

### Debug 檔案
已輸出至 debug-{changeName}-{timestamp}.md

### 選項
1. 人工介入修復
2. 調整 spec/design 後重跑
3. 其他
```

---

## Guardrails

- 每個 agent 的 prompt 只傳變更名稱和目錄，讓 agent 自行讀取 artifacts
- 不在 prompt 中貼入檔案內容
- Coder 的輸出（檔案清單 + 設計決策）由 orchestrator 保留，用於傳遞給後續 agent 和 retry
- Retry 迴路中，附帶前一輪的輸出摘要，避免 context 斷裂導致重複犯錯
- 同歸屬的 WARNING 合併為一個修復任務，減少不必要的來回
- 獨立的修復任務可平行派發
- 3 輪上限後必須停下來問人，不可繼續嘗試
- Pipeline 開始前確保在功能分支上，不直接在主要分支開發
- Pipeline 完成後不自動 commit，等人工驗收通過後再走交付流程
- WARNING re-check 使用 Sonnet 而非 Opus，存在品質判斷閾值差異的風險——Sonnet 可能將 subtle issue 判定為已修復。若同一 WARNING 在 re-check 後人工驗收時再次出現，應考慮升級為 Opus 完整 review
