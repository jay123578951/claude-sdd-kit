---
name: code-fix
description: Tier 2 輕量 Pipeline — Coder + Tester 快速修復小改動，含 Spec 影響檢查
---

Tier 2 輕量版 Agent Pipeline，適用於不需要設計決策的小改動（跨檔案 bug fix、小型 UI 調整、composable 修正）。

與 `/code:feat`（Tier 3）的差異：
- 不依賴變更 artifact — 需求從對話描述取得
- 只派發 Coder + Tester（不跑 Reviewer）
- 完成後走 Spec 影響檢查（非完整 change 歸檔流程，如 /opsx:verify / sync / archive）

**Input**: 對話中已描述問題或需求。未描述清楚時，先釐清再啟動。

---

## 適用判斷

在使用前確認符合 Tier 2：

| 條件 | 要求 |
|------|------|
| 影響檔案數 | 2-5 個 |
| 需要設計決策 | 否 |
| 新增 API / 元件 | 否 |

不符合 → Tier 1（直接改）或 Tier 3（`/code:feat`）。

---

## 專案配置

### Agent Knowledge Skills

| Agent | Skills（必載） | 可選 Skills | 用途 |
|-------|---------------|------------|------|
| Coder | `vue`, `vue-best-practices`, `nuxt`, `antfu` | 由 orchestrator 根據問題內容預判並寫入 prompt（如 `pinia`, `unocss`, `vite`, `vue-router-best-practices`, `vueuse-functions`, `pnpm`, `turborepo`） | Vue/Nuxt 開發慣例、程式碼風格、元件拆分守則（與 `/code:feat` 同步） |
| Tester | `vitest`, `antfu`, `vue-testing-best-practices` | — | 測試框架用法、Vue 元件測試慣例 |

### Model 策略

| Agent | Model |
|-------|-------|
| Coder | sonnet（預設）/ opus |
| Tester | sonnet |
| 註解整理 | sonnet |

Coder 預設 sonnet。Tier 2 依定義為單模組、無設計決策的小改動，故 `/code:feat` 的「架構變更」「設計決策密集」升級條件在此不適用；僅保留下列兩條升級規則：

- **首次派發**：改動觸及安全敏感路徑（auth、payment、API key 處理、session 管理）→ `{coderModel}` 設為 `opus`
- **Retry 動態升級**：Coder ↔ Tester retry 進入第 2 輪起，`{coderModel}` 升為 `opus`（後續修復皆用 opus，不再降回）。連 2 輪未過代表非表層瑕疵

判定保守。一般小改動維持 sonnet。

---

## 流程

### Step 1: 整理需求

從對話中擷取：

1. **問題描述**：什麼壞了 / 要改什麼
2. **預期行為**：修好後應該怎樣
3. **可能影響的檔案**：根據問題描述搜尋定位

宣告：「Tier 2 code:fix：{問題摘要}」

**Coder Skills 預判**

Orchestrator 根據問題描述和相關檔案判斷 Coder 除了必載的 `vue`, `vue-best-practices`, `nuxt`, `antfu` 外，還需要哪些額外 skills，寫入 `{additionalSkills}` 變數（觸發規則與 `/code:feat` 一致，避免兩個 pipeline 走出不同風格）：
- 問題涉及 store / state → 加 `pinia`
- 問題涉及 CSS utility / atomic → 加 `unocss`
- 問題涉及 build config / plugin → 加 `vite`
- 問題涉及路由 / middleware / navigation guard / route params → 加 `vue-router-best-practices`
- 問題涉及 DOM 事件 / 瀏覽器 API / 常見 composable 場景（local storage、media query、resize、clipboard 等）→ 加 `vueuse-functions`
- 專案使用 pnpm（`package.json` 含 `"packageManager": "pnpm@..."` 或根目錄存在 `pnpm-lock.yaml`）且問題涉及依賴 / workspace / catalog / patch → 加 `pnpm`
- 專案為 monorepo（根目錄 `Glob turbo.json` 命中）且問題跨 package → 加 `turborepo`

### Step 1.5: 建立工作分支

確認當前 git 分支狀態：

- 若在 `main`、`develop` 等主要分支上 → `git checkout -b fix-<描述>`
- 若已在功能分支上 → 跳過

### Step 2: 派發 Coder Agent

依「Model 策略」判定 `{coderModel}`（首次派發預設 sonnet，安全敏感路徑升 opus）。使用 Task tool 派發 subagent（model: {coderModel}）：

```
你是 Coder Agent。

開始工作前：
1. 用 Skill tool 載入以下 skills：vue, vue-best-practices, nuxt, antfu{additionalSkills}
2. 讀取專案的 CLAUDE.md 了解專案慣例

注意：若修復過程中發現需要用到上方未列出的 skill，可自行載入補充。

問題描述：
{從對話擷取的問題描述}

預期行為：
{修復後的預期結果}

可能相關的檔案：
{列出定位到的檔案路徑}

請修復此問題：
- 遵循專案設計系統與慣例
- 善用既有的 composables 和 utils
- 只修改必要的部分，不做額外重構

完成後跑專案 lint script 確保代碼通過 lint（優先用專案既有 script，如 `pnpm lint --fix`；無對應 script 才 fallback 到 `pnpm exec eslint --fix`。依專案 package manager 調整指令，不要用裸 `npx`）。

輸出：
1. 修改的檔案路徑與變更摘要
2. 修復邏輯的簡要說明（供 retry 時作為上下文參考）
```

### Step 3: 派發 Tester Agent

使用 Task tool 派發 subagent（model: sonnet）：

```
你是 Tester Agent。

開始工作前：
1. 用 Skill tool 載入以下 skills：vitest, antfu, vue-testing-best-practices
2. 讀取 Coder 修改的檔案：
   {Coder 回報的檔案路徑列表}

預期行為：
{修復後的預期結果}

撰寫測試：
- 測試檔放在與源碼同目錄（foo.test.ts）
- 使用 describe/it API 結構
- 針對修復的行為撰寫測試，確保 bug 不會再現
- 純邏輯函式應抽出為獨立模組，測試 import 實際模組（不複製邏輯）

排除規則（不要撰寫以下測試）：
- TypeScript 型別/介面欄位存在性測試（TypeScript 編譯器已保證型別正確性）
- 無法 import 實際模組時（如 Nuxt composable），跳過該模組的單元測試，不要複製邏輯自測

完成後執行測試：優先跑專案 test script（如 `pnpm test`，依專案 package manager 調整）；需要逐項失敗資訊時用 `pnpm exec vitest run --reporter=verbose`。不要用裸 `npx`。

輸出：
1. 建立的測試檔案路徑
2. 測試結果（通過/失敗）
3. 若有失敗，列出每個失敗的測試名稱和錯誤原因
4. 無法測試的模組清單（列出模組名稱與跳過原因，如「useWeatherApi — Nuxt composable，無法在單元測試中 import」）
```

**測試失敗時**：進入 Retry 迴路（見下方）。

### Step 4: 報告結果

顯示完成摘要，提示人工確認修復結果。

### Step 4.5: 註解整理（Sonnet subagent）

Coder/Tester settle 後、Spec 影響檢查前，派發一次註解整理 Agent 清除本次修復累積的過時／思考流程／冗餘註解。

- 載入 `code-comment` skill 取得整理規範與輸出格式
- 使用 Task tool 派發 subagent，固定 **`subagent_type: general-purpose` + `model: sonnet`**
- scope 為「本次修改的檔案清單」（Coder 產出 + Tester 測試檔），由 orchestrator 注入 prompt 的 `{changedFiles}`
- 整理 Agent 依守則**直接套用 Edit**並自跑 lint --fix（指令依專案偵測，優先 `pnpm lint --fix`，不寫死 `npx`；不可刪除功能型指令註解，如 `eslint-disable`、`@ts-expect-error`、`istanbul ignore`、`v-html` 安全註記）
- 整理完成後 orchestrator **重跑測試**（專案 test script，如 `pnpm test`）作為安全網；失敗回整理 Agent 修正（最多 1 輪），仍失敗 → 停下來問人

### Step 5: Spec 影響檢查

執行 Spec 影響檢查：

```
1. 列出本次修改的所有檔案
2. 比對以下位置的相關規格：
   - openspec/specs/    （OpenSpec 主規格庫）
   - 專案 CLAUDE.md 中定義的其他設計文件位置
3. 判斷：
   ├─ 無影響 → 在完成摘要中標記「Spec 無影響」
   └─ 有影響 → 更新對應 spec，在完成摘要中列出已更新的 spec
```

**不執行 commit。** Commit 時機由人工決定（通常在 change 歸檔時一併處理）。

---

## Retry 迴路

### 測試失敗 → Coder 修復

1. 將以下資訊傳給 Coder Agent：
   - 失敗的測試名稱和錯誤訊息
   - **前一輪 Coder 的輸出摘要**（修改的檔案清單 + 修復邏輯說明）
   - 明確的修復指示
2. Coder prompt 須明確指示：**不需重讀 CLAUDE.md 與周邊檔案**，只讀前輪輸出摘要、失敗報告、要修改的檔案（節省重複 context 載入）
3. **第 2 輪起 `{coderModel}` 升為 `opus`**（見 Model 策略），後續修復派發皆用 opus
4. Coder 修復後，重新派發 Tester Agent 驗證
5. 最多 3 輪，修不好 → 停下來問人

### Retry 計數

- ESLint 錯誤：Coder 自行修復，不計入重試次數
- 測試失敗：Coder ↔ Tester 最多 3 輪
- 3 輪修不好 → 停下來問人（問題可能較嚴重或 AI 忽略關鍵細節）

---

## 輸出格式

### 完成

```
## Tier 2 完成：{問題摘要}

### Agent Pipeline 結果
- Coder: ✓ 完成（N 個檔案）
- Tester: ✓ 通過（M 個測試）
- 註解整理: ✓ 清除 X 處 / 改寫 Y 處（測試重跑通過）

### Pipeline 統計
- Coder 派發次數：{coderCalls}（含 retry）
- Tester 派發次數：{testerCalls}（含 retry）

### Retry 記錄
（若有 retry，列出每輪的問題與修復摘要）

### Spec 影響檢查
- [受影響的 spec 列表與更新摘要] 或「無影響」

### 下一步
請人工確認修復結果。Commit 時機由人工決定。
```

### 遇到阻塞

Orchestrator 在宣告阻塞前，輸出 debug 檔案 `debug-fix-{timestamp}.md`（放在專案根目錄）：

```markdown
## Pipeline 暫停：{問題摘要}

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
## Pipeline 暫停：{問題摘要}

### 問題
{問題描述}

### 已完成
- Coder: ✓
- Tester: ✗ 第 3 輪仍失敗

### Pipeline 統計
- Coder 派發次數：{coderCalls}
- Tester 派發次數：{testerCalls}

### Debug 檔案
已輸出至 debug-fix-{timestamp}.md

### 選項
1. 人工介入修復
2. 重新描述問題後重跑
3. 升級為 Tier 3（建立 OpenSpec change，走 /code:feat）
```

---

## Guardrails

- Coder prompt 直接描述問題，不依賴變更 artifact
- 不在 prompt 中貼入檔案內容，讓 agent 自行讀取
- Coder 的輸出（檔案清單 + 修復邏輯）由 orchestrator 保留，用於傳遞給 Tester 和 retry
- Retry 迴路中，附帶前一輪的輸出摘要，避免 context 斷裂導致重複犯錯
- 3 輪上限後必須停下來問人，不可繼續嘗試
- 註解整理在 Tester settle 後執行，只動註解不動 code 邏輯，整理後必重跑測試作為安全網
- Pipeline 完成後不自動 commit，等人工確認
- Spec 影響檢查不可跳過 — SDD 核心原則
