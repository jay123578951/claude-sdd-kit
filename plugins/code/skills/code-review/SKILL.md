---
name: code-review
description: Use when reviewing code changes for quality, security, and project conventions — standalone or as review standard within code-feat pipeline
---

獨立的 Code Review skill，定義 review 維度、流程與輸出格式。透過 Task tool 派發 **Opus Reviewer Subagent** 執行 review，提供與主對話 Sonnet 隔離的獨立視角與深度推理。可在任何場景獨立呼叫，也作為 `code-feat` Reviewer 的 review 標準單一來源。

**Input**: 可選指定 review 範圍（見下方模式）。未指定時自動偵測 git diff。

**Model 策略**：Reviewer 走 Opus subagent，與主對話的 Sonnet 隔離（context 不互相污染、避免自評自審），同時取得 Opus 的深度推理能力。

**成本提示**：Opus subagent 每次呼叫會消耗較多 Claude token。以下情境建議略過此 skill，直接請主對話讀檔給意見：

- 改動範圍僅 1–2 個小檔案
- 單純格式/文字修正（ESLint/TypeScript 已能捕捉）
- 想要快速 sanity check 而非正式 review

---

## Review 模式

根據輸入自動判斷：

```
/code:review                     → 自動偵測：git diff 未 commit 的變更
/code:review --staged            → 只看 staged changes
/code:review --branch feat/xxx   → 整個 branch 相對 main 的 diff
/code:review --change xxx        → 讀取 OpenSpec 變更 artifact 作為 review 基準
```

被 `code-feat` 的 Reviewer 載入時，由呼叫方 prompt 指定範圍，不需自動偵測。

---

## 流程

### Step 1: 準備

1. 讀取專案的 CLAUDE.md 了解專案慣例
2. 依 review 模式確認變更範圍：
   - 自動偵測：`git diff` + `git diff --staged`
   - `--staged`：`git diff --staged`
   - `--branch`：`git diff main...<branch>`
   - `--change`：讀取 `openspec/changes/<name>/` 下的所有 artifacts

### Step 2: 判斷是否升級為 adversarial 模式

下列任一條件成立即升級為 adversarial（主動找漏洞、red team 視角）：

- 安全敏感路徑（auth、payment、API key 處理、session 管理）
- 資料庫 schema 變更或生產資料遷移
- 使用者明確要求深度 review
- `code-feat` Reviewer 在第 2 輪 retry 仍 FAIL 時升級

被 `code-feat` 載入時，adversarial 與否由呼叫方指定。

### Step 3: 派發 Reviewer Subagent

使用 Task tool 派發 subagent，派發參數固定為 **`subagent_type: general-purpose` + `model: opus`**。prompt 模板見「Reviewer Subagent Prompt 模板」章節，依模式注入對應 scope 與 adversarial flag。

Subagent 自行讀檔、自行整合 findings、直接輸出本 skill 定義的「輸出格式」最終報告。主對話收到後直接呈現，不需要再做重新包裝或補充檢查（這些責任已在 prompt 中明確交給 subagent）。

被 `code-feat` Reviewer 載入時，由呼叫方 orchestrator 直接派發 Opus subagent，並把 Coder/Tester 產出摘要、變更名稱注入 prompt。

**Context budget 守則**（派發前 orchestrator 必算）：

1. 計算改動檔案數與總行數（git diff 行數，含改動本身）
2. 若 **改動涉及 < 10 個檔案 且 總行數 < 3000**：subagent 讀取所有改動檔案完整內容
3. 若 **超過上述閾值**：orchestrator 在派發 prompt 中明確列出兩類檔案：
   - **完整讀取**：改動最密集、結構性改動最大的檔案（建議 ≤ 10 個檔，總行數 ≤ 3000）
   - **僅讀 diff**：其餘檔案，subagent 只看 diff 與簡短的函式定義列表
4. Subagent 在執行中若仍發現 context 不足，須在報告開頭以 `open question` 明確標記受影響的檔案，繼續完成其餘檢查

**Subagent 執行失敗 / 派發異常時**（獨立模式與 `code-feat` 模式統一紀律）：

記錄錯誤並停下來問人，**不退化為主對話 Sonnet 自做 review**——這會破壞「Sonnet 寫的 code 由獨立 Opus subagent 審」的設計初衷。`code-feat` 流程下，orchestrator 將整體狀態（含已完成的 Coder/Tester 產出、Reviewer 派發失敗原因）報告給使用者後等待後續指示，不繼續推進 retry 迴路。

### Step 4: 呈現結果

主對話將 Reviewer Subagent 的輸出原樣呈現給使用者，並做最後的 STOP 控制。

**STOP 規則**：呈現 findings 後立即停止。不修復任何問題。必須明確詢問使用者要修什麼才能動檔案。

---

## Reviewer Subagent Prompt 模板

本章節是 Review 維度、嚴重程度、判定規則、輸出格式的 **single source of truth**——所有規範均內嵌於下方 prompt 模板。文件其他章節（如「Targeted Check 模式」、「與 Pipeline 的關係」）僅描述外圍流程，不另存規範副本。

呼叫方依模式組裝 prompt 並用 Task tool 派發。

**派發參數**：

- `subagent_type`: `general-purpose`
- `model`: `opus`
- `prompt`: 下方模板展開後的字串（**不可原樣帶入**）

**模板展開規則**（派發前必做）：

1. `{變數}` 佔位符（如 `{changeName}`、`{coderOutputFiles}`、`{standard | adversarial}`、`{auto | staged | branch:<name> | change:<changeName>}`）一律以實際值替換為單一具體字串
2. `{若...：}` 條件區塊：條件成立時刪掉條件標頭、保留下方內文；條件不成立時整段刪除（含底下所有內容，直到遇到下一個 `---` 分隔線或下一個 `{若...：}` 標頭為止）
3. 未展開的條件標頭被字面送進 subagent 會導致 review 邏輯混淆——派發前必須目視確認模板已展開乾淨，不應留下任何 `{若...：}` 或 `{a | b}` 字串

通用 prompt 模板：

```
你是 Code Reviewer Agent，使用 Opus 深度推理執行獨立 code review。

模式：{standard | adversarial}
Scope：{auto | staged | branch:<name> | change:<changeName>}
{若為 change 模式：}
變更名稱：{changeName}
變更目錄：openspec/changes/{changeName}/

---

開始工作前：

1. 讀取專案的 CLAUDE.md，理解專案慣例（特別注意：繁體中文 UI 文字、CSS 變數使用、設計系統慣例）
2. 取得變更內容：
   - auto: `git diff` 與 `git diff --staged`
   - staged: `git diff --staged`
   - branch: `git diff main...<branch>`
   - change: 先讀取 openspec/changes/{changeName}/ 下的 proposal.md、design.md、tasks.md、specs/ 全部 artifacts，再讀取 git diff（working tree 對 main）
3. 讀取被修改的檔案：
   - 預設：所有改動檔案讀完整內容（不只看 diff，需要 context 才能判斷重複邏輯、過度抽象、結構性問題）
   - 若 orchestrator 已在 prompt 中標註「僅讀 diff」清單，依標註執行（context budget 守則的 fallback 路徑）
   - 執行中若仍發現 context 不足，於報告開頭以 `open question` 標記受影響的檔案，不要中斷
{若 code-feat 載入：}
4. 讀取本 prompt 末段附帶的 Coder/Tester 產出摘要

---

依照下方「Review 維度」逐項檢查並標記 finding 嚴重度：

### 必檢

| 維度 | 檢查項目 |
|------|---------|
| 程式碼品質 | 命名一致性、函式/元件結構、可讀性、重複邏輯、過度抽象 |
| 安全性 | API key 暴露、XSS、注入風險、敏感資料洩漏 |
| 專案慣例 | 繁體中文 UI 文字、CSS 變數使用（不硬編碼顏色/間距/字體大小）、遵循專案設計系統慣例、CLAUDE.md 中定義的其他規則 |

### 條件檢查

| 維度 | 觸發條件 | 檢查項目 |
|------|---------|---------|
| 測試品質 | 變更含測試檔案 | 測試是否有效驗證行為（非複製邏輯自測）、排除規則是否遵守 |
| Spec 一致性 | change 模式 | 所有 requirements 是否實作、所有 scenarios 是否覆蓋、設計決策是否遵循 design.md |

### 不檢查

- TypeScript 型別正確性（編譯器負責）
- ESLint 能捕捉的格式問題（lint 負責）
- 效能最佳化（除非有明顯的 O(n²) 或記憶體洩漏）

---

{若 adversarial：}

此次為 **adversarial / 深度對抗** review：

- 主動找漏洞，不假設輸入合法
- 從攻擊者視角分析每個邊界：輸入驗證、權限、race condition、injection、序列化/反序列化、整數溢位、TOCTOU
- 質疑每個 happy path 假設，列出 fail-open 與權限提升路徑
- 對 retry 已升級的情境，要特別說明前一輪為何被遺漏、本輪如何補強
- 列出非明顯的失敗模式（不只「可能 X」，要給檔案:行號的證據）

---

Grounding rules：

- 每個 finding 必須指到明確的「檔案:行號」
- requirement / scenario 未實作時必須引用 spec 原文作為證據
- 不要推測「可能的問題」，只報告有證據支持的事實
- 區分 observed fact 與 inference，inference 必須明確標註
- 若資訊不足，標記為 open question 繼續完成其餘檢查，不要詢問澄清問題

---

依下方格式輸出最終報告（直接是最終格式，不需要主對話後續包裝）：

## Code Review：{scope 描述}

### 判定：PASS / PASS with WARNING / FAIL

判定規則：
- 有任何 CRITICAL → FAIL
- 有 WARNING 但無 CRITICAL → PASS with WARNING
- 只有 SUGGESTION 或無問題 → PASS

{若為 change 模式追加：}

### Spec Alignment 檢核表

| Requirement | 狀態 | 證據 |
|-------------|------|------|
| ... | 已實作 / 部分實作 / 未實作 | 檔案:行號 或 spec 原文 |

### 問題清單

| # | 嚴重度 | 歸屬 | 檔案:行號 | 描述 | 修復建議 |
|---|--------|------|----------|------|---------|
| 1 | CRITICAL | coder | foo.vue:42 | ... | ... |
| 2 | WARNING | tester | foo.test.ts:10 | ... | ... |

嚴重度定義：
- CRITICAL：會導致功能錯誤、安全漏洞、或資料遺失 → 必須修復
- WARNING：違反專案慣例、影響可維護性、或潛在風險 → 需要修復
- SUGGESTION：可改善但不影響正確性 → 僅記錄

歸屬：
- coder：實作代碼問題
- tester：測試代碼問題

按嚴重度排序（CRITICAL → WARNING）。

### SUGGESTION

（僅記錄，不要求修復）

- foo.vue:15 — ...
- bar.ts:30 — ...

### 摘要

{1-2 句整體評價}
```

---

## Targeted Check 模式

當被用於 `code-feat` 的 WARNING re-check 時，執行精簡版 review。**目的是驗證單一輪 WARNING 修復是否正確，不重新做完整 review**。

### 設計重點

- **由 Sonnet subagent 直接讀檔執行，不派發 Opus**（避免為小範圍 re-check 消耗較貴的 Opus quota，並節省派發開銷）
- 只讀取改動的檔案和對應行數
- 只驗證原始 WARNING 是否已正確修復、是否引入新問題
- 不重新掃描所有檔案
- 輸出格式沿用上方模板的「依下方格式輸出最終報告」段，但 scope 描述標記為「targeted re-check」
- **不計入 Reviewer retry counter**（targeted re-check 屬於 WARNING 修復驗證，與 FAIL retry 是不同性質）

### 派發參數

- `subagent_type`: `general-purpose`
- `model`: `sonnet`
- `prompt`: 下方模板展開後的字串

### Targeted Check Prompt 模板

模板展開規則同上方 Reviewer Subagent Prompt 模板（`{變數}` 替換、`{若...：}` 條件區塊整段處理）。

```
你是 Code Reviewer Targeted Check Agent，使用 Sonnet 對前一輪 WARNING 修復做精簡 re-check。

前一輪 Review 報告中的 WARNING 清單：
{warningList}

修復涉及的檔案：
{fixedFiles}

---

開始工作前：

1. 讀取上方列出的修復檔案——只讀改動行數及其前後必要的 context（不需讀完整檔）
2. **不要**重新掃描其他檔案、不要讀取變更 artifact 或 design.md
3. **不要**對未被修復的部分做 review；不要找新的 finding 來「補強」報告

逐項驗證每個原始 WARNING：

1. 該 WARNING 是否已正確修復？
2. 修復是否引入新問題？（同一檔案、同一行附近）

---

Grounding rules：

- 每個結論必須指到「檔案:行號」
- 如果原始 WARNING 已修復且未引入新問題：標 `已修復`
- 如果原始 WARNING 未正確修復（修補不到位、改錯方向）：標 `未修復` + 簡述為何不到位
- 如果修復引入新問題：標 `修復引入新問題` + 證據
- 不主動找新 finding；若發現非原 WARNING 範圍的問題，僅在「附帶觀察」段以 SUGGESTION 列出

---

輸出格式：

## Code Review：targeted re-check ({warningCount} 個 WARNING)

### 判定：PASS / FAIL

判定規則：
- 所有原始 WARNING 皆 `已修復` 且無 `修復引入新問題` → PASS
- 任一原始 WARNING `未修復` 或有 `修復引入新問題` → FAIL

### 驗證清單

| # | 原始 WARNING | 狀態 | 證據 |
|---|-------------|------|------|
| 1 | {原始 WARNING 摘要} | 已修復 / 未修復 / 修復引入新問題 | 檔案:行號 + 簡述 |

### 附帶觀察（選填）

- 檔案:行號 — SUGGESTION 級別的觀察

### 摘要

{1 句整體評價}
```

---

## 與 Pipeline 的關係

| 使用者 | 如何使用 |
|--------|---------|
| `code-feat` Step 6 | Orchestrator 派發 Opus Reviewer subagent，載入此 skill 取得 review 標準 |
| `code-feat` WARNING re-check | Sonnet subagent 載入此 skill，執行 targeted check |
| `code-fix`（可選） | 完成後人工決定是否跑 `/code:review` |
| 獨立使用 | 任何時候對任意 diff 執行 review |

此 skill 只負責「怎麼 review」與「派發給誰」。失敗派發邏輯（retry、歸屬路由、3 輪上限）由呼叫方（`code-feat` 或人）管理。
