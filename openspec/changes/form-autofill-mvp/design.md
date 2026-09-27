## Context / 背景

本專案起始於 `reference/auto-fill-html_forms.md` 的一份初版程式碼：五個檔案（`manifest.json`、`popup.html`、`popup.js`、`content.js`、`background.js`），由 `background.js` 直接以硬編碼的金鑰呼叫 LiteLLM。該版本證明了核心資料流可行，但存在四類工程缺陷，已在 `reference/rules_20260724.md` v1.1 中逐條列為禁止行為：

1. **金鑰硬編碼**（R6.4）— `LITELLM_API_KEY` 直接寫在 `background.js`。
2. **解析失敗靜默**（R18.1）— `catch` 內 `return {}`，使用者看到「處理完成」但欄位全空。
3. **僅設 `el.value` 不派發事件**（R17.3）— React / Vue 等框架不會更新狀態。
4. **未排除 `input[type=file]`**（R17.2）— 瀏覽器禁止程式寫入其 value，產生永遠填不成功的假成功。

本變更在該基礎上建立可驗證、可維護的 MVP，並依使用者決策將 LLM 呼叫移至自建後端（LiteLLM 作為後端所呼叫的模型端點），使金鑰不再進入擴充功能。

本變更跨四個能力（`extension-ui`、`content-script`、`background-worker`、`backend-api`）並引入一個新的相依（Express），且涉及資料離開瀏覽器的安全性議題，因此需要本設計文件。

The project starts from the initial code in `reference/auto-fill-html_forms.md`: five files driven by `background.js` calling LiteLLM directly with a hardcoded key. That version proved the core data flow works but carries four engineering defects, each now codified as a prohibition in `rules_20260724.md` v1.1: hardcoded credentials (R6.4), silent `{}` on parse failure (R18.1), setting `el.value` without dispatching events (R17.3), and failing to exclude `input[type=file]` (R17.2).

This change productizes that into a verifiable, maintainable MVP, and — per the user's decision — moves the LLM call behind a self-hosted backend so no credential enters the extension.

This change spans four capabilities, introduces one new dependency (Express), and involves page data leaving the browser. It therefore warrants a design document.

---

## Goals / Non-Goals

**Goals / 目標**

- 交付一條端到端可運作的資料流：Popup 觸發 → Content Script 掃描 → Service Worker 組件 → 後端 → LiteLLM → 回填 → 逐欄摘要。
  Deliver a working end-to-end flow: popup trigger → content script scan → service worker assembly → backend → LiteLLM → fill → per-field summary.
- 讓所有失敗路徑在使用者可見的摘要中呈現，且不存在「LLM 有回應但欄位沒填」的無聲失敗。
  Make every failure path visible in the user-facing summary, with no silent "the LLM replied but nothing was filled" failure.
- 確保 LiteLLM 金鑰只存在於後端環境變數，永不進入擴充功能或 Git。
  Keep the LiteLLM credential exclusively in backend environment variables, never in the extension or Git.
- 讓識別碼不重複、選項白名單、欄位約束與敏感欄位保護四項規則在回填前全部生效。
  Enforce unique identifiers, the option whitelist, field constraints, and sensitive-field protection before any value is written.
- 鎖定跨能力欄位結構契約的單一權威定義位置，避免兩份 spec 各自演化（R3）。
  Establish a single authoritative location for the cross-capability field contract so two specs cannot drift (R3).

**Non-Goals / 明確不做**

- **不支援 iframe 內的表單**。跨 iframe 需要處理 `chrome.scripting.executeScript` 的 frameId 與跨來源限制，屬獨立議題。
  No form support inside iframes; that requires frame-id and cross-origin handling and is a separate concern.
- **不主動送出表單**。永不呼叫 `form.submit()` 或點擊送出鈕（R6.7）。
  Never auto-submit; never call `form.submit()` or click a submit button (R6.7).
- **不支援多頁籤並行批次填寫**。
  No multi-tab batch filling.
- **不做深度的框架受控元件整合**。以 `input` / `change` 事件達成相容為限，不解析 React 的 value tracker。
  No deep framework controlled-component integration; event-based compatibility only, without reverse-engineering React's value tracker.
- **不導入 schema 驗證函式庫**（zod / ajv）與 TypeScript 建置流程。兩者皆為待確認事項（Q2、Q3），MVP 先以手動驗證函式達成同一行為。
  No schema validation library and no TypeScript build step; both are open questions (Q2, Q3) and the MVP achieves the same behaviour with hand-written validation.
- **不實作 `PRD.md`**。依 R3，`openspec/specs/` 為權威來源，`PRD.md` 待規格穩定後才撰寫。
  No `PRD.md`; per R3 the specs are authoritative and the PRD comes later.

---

## Decisions / 設計決策

### D1 — LLM 呼叫放在後端，而非 Service Worker 直呼

**決策 / Decision**：擴充功能呼叫自建 Node.js + Express 後端，由後端呼叫 LiteLLM。擴充功能不直接連線任何 LLM 端點。

**理由 / Rationale**：使用者已明確選擇保留後端。此決策同時解決三個問題：金鑰不進擴充功能（R6.4）、可在後端統一驗證與過濾（R18.1 / R18.3）、未來若需加入 AI 味後處理或速率控制不需重寫擴充功能。

**替代方案 / Alternatives considered**：Service Worker 直呼 LiteLLM（即初版做法）。優點是少一個服務、無 CORS 問題；缺點是金鑰必須進擴充功能，任何使用者都能從原始碼取出，違反 R6.4。

**代價 / Cost**：使用者必須自行啟動後端服務，MVP 的使用步驟比初版多一步。

### D2 — 金鑰存放位置：後端環境變數，擴充功能只存非敏感設定

**決策 / Decision**：`chrome.storage.local` 僅存放後端 base URL 與模型名稱。`LITELLM_API_KEY` 僅存在後端 `.env`（已由 `.gitignore` 排除）。

**理由 / Rationale**：這是唯一能同時滿足「使用者在 Options 頁設定」與「金鑰不進擴充功能」的切分。使用者設定的是*連線資訊*，不是*憑證*。

**替代方案 / Alternatives considered**：(a) 金鑰存 `chrome.storage.local` — 仍可被其他腳本或使用者本人讀取，等同公開，不符合 R6.4。(b) 硬編碼於 `background.js` — 即初版做法，永久外洩且難以輪替。

### D3 — 欄位識別碼的三級優先序與去重

**決策 / Decision**：識別碼優先序為 `id` → `name` → 注入 `data-ai-id="ai_input_<index>"`；當兩個欄位解析出相同識別碼時，後者改用注入值。

**理由 / Rationale**：`id` 最穩定；`name` 對 radio 群組有意義但會重複，故需去重規則；注入 `data-ai-id` 保證唯一。回填時以 `getElementById` → `[name="..."]` → `[data-ai-id="..."]` 三級反查，與指派順序一致。

**替代方案 / Alternatives considered**：只用 DOM index — 頁面若動態重排元素，index 會指向錯誤欄位，屬危險做法。改用 CSS selector 路徑 — 複雜且易受 class 命名影響。

### D4 — 敏感欄位以「欄位名稱 / label 關鍵字」判定，MVP 先用規則式

**決策 / Decision**：MVP 以欄位的 `id`、`name`、`questionText` 比對一份關鍵字清單（身分證、護照、信用卡、CVV、銀行帳號、密碼、病歷等）判定敏感欄位，命中即留空並記入 `skipped`。

**理由 / Rationale**：規則式可預測、可測試、無額外相依，符合 MVP 定位。Q1 已登記為待確認事項，ML / 語意分類留待有實際誤判率數據後再評估。

**替代方案 / Alternatives considered**：ML / 語意分類 — 覆蓋率可能更好，但引入模型相依且難以測試。完全不保護 — 直接違反 R6.6，本專案最高風險。

**已知不足 / Known limitation**：關鍵字清單會漏掉用詞 unusual 的敏感欄位。緩解見 R3 風險項目。

### D5 — 敏感欄位清單在 content script 與 backend 雙端各做一次

**決策 / Decision**：後端依提交的欄位結構丟棄敏感欄位答案並記入 `skipped`；content script 再依同一份關鍵字清單二次防護，不填入任何值。

**理由 / Rationale**：縱深防禦（R16 思維）。後端可能是舊版或被繞過，content script 的二次檢查確保敏感欄位在瀏覽器端無論如何都不會被寫入。兩端使用同一份關鍵字清單，以單一模組（`backend/validate.js` 為權威）供參考。

**替代方案 / Alternatives considered**：只在後端過濾 — 若使用者同時載入舊版 content script，敏感值仍可能進入 DOM。單一責任點較乾淨但缺乏縱深。

### D6 — 以 `skipped` 陣列取代靜默丟棄

**決策 / Decision**：所有被丟棄的答案（選項不符、超長、敏感、識別碼無對應）一律記入 `skipped` 並附原因碼，由 Popup 逐項顯示。

**理由 / Rationale**：R6.7 要求逐欄摘要；`skipped` 是讓「沒有填滿」變得可見的唯一資料結構。放棄填入的欄位必須有名有姓地被看見，否則使用者會誤以為表單已填完整而直接送出。

**原因碼 / Reason codes**：`sensitive_field`、`option_not_found`、`max_length_exceeded`、`no_matching_element`（content script 端產生）。

### D7 — 事件派發而非直接操作框架狀態

**決策 / Decision**：填值後一律派發 `input` 與 `change` 事件（`bubbles: true`）。

**理由 / Rationale**：React 的受控元件以合成事件監聽變更；`el.value = x` 不會觸發其 onChange。派發事件是跨框架的最低公分母做法，成本極低（R17.3）。

**替代方案 / Alternatives considered**：使用 `Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set.call(el, v)` 繞過 React 的 value tracker — 能強制觸發，但依賴框架內部實作細節，版本升級易碎，且不涵蓋 Vue / Svelte。列為未來可能的加強，非 MVP 範圍。

### D8 — 後端以四個檔案拆分職責

**決策 / Decision**：`server.js`（路由與啟動）、`prompt.js`（Prompt 組裝）、`llm.js`（LiteLLM 客戶端）、`validate.js`（回應驗證）。

**理由 / Rationale**：R8 明確要求後端檔案至少如此拆分。Prompt 模板與 HTTP 路由若同檔，調 Prompt 時會被迫觸碰路由程式碼。

---

## Risks / Trade-offs / 風險與取捨

**[R-1] 頁面欄位文字（含個資）離開瀏覽器送往第三方模型 →**
頁面文字會經後端送往 LiteLLM 供應商。使用者可能不預期此資料流。
緩解：`pageUrl` 只送 scheme + host，剝除查詢字串權杖（`backend-api` 規格已定義此行為）；僅送欄位結構描述，不送完整 HTML、cookie 或整頁文字（`backend-api` 規格）；在 Options 頁以明確文案揭露資料流向，讓使用者能在知情下決定是否啟用；R18.4 將此列為禁止行為。
Page-derived field text passes through the backend to the LLM provider. Mitigation: strip query-string tokens from `pageUrl`, send only the field structure, disclose the data flow in the Options page, and codify it as a prohibition in R18.4.

**[R-2] 敏感欄位關鍵字清單會漏判 →**
使用者若把身分證欄位命名為「證件號碼」而非清單中的「身分證字號」，該欄位會被填入幻覺值，而這正是 R6.6 要防止的情況。
緩解：MVP 將關鍵字清單設計為易於追加的常數陣列並於 R14 列為變更時必須同步的文件項目；清單缺漏視為已知限制而非缺陷，Q1 啟動後以實際誤判率決定是否升級為語意分類。**本項為 MVP 已知的資料正確性風險，應在 Options 頁告知使用者逐欄檢視。**
The keyword list can miss sensitive fields with unusual names, which is exactly the harm R6.6 exists to prevent. Mitigation: keep the list as an easily extensible constant array registered in R14 as a must-sync document; treat misses as a known MVP limitation, not a defect, and let Q1 decide on semantic classification once real miss rates exist. **This is a known data-correctness risk; the Options page should tell users to review every field.**

**[R-3] 使用者未啟動後端，Popup 按鈕無效 →**
MVP 需要使用者自行 `npm start` 啟動後端，對非技術使用者是明顯門檻。
緩解：Popup 在設定缺失與連線失敗時給出具體可行動的訊息（指明啟動指令或指出後端未啟動），而非「處理完成」或空白；R18.5 將此列為禁止行為。長期緩解為將後端部署為雲端服務，屬本專案後續議題。
Users must start the backend themselves. Mitigation: concrete actionable error messages naming the startup step, never a bare failure; codified in R18.5. Longer-term mitigation is cloud deployment, a later concern.

**[R-4] `input` / `change` 事件對部分框架仍不足 →**
少數使用非標準元件的頁面（自製 framework、canvas 內輸入、部分日期選擇器）可能仍不會同步狀態，導致「值看得到但送出為空」。
緩解：R17.3 已將事件派發定為強制；此類頁面屬已知限制，MVP 不追求 100% 覆蓋。若使用者回報特定網站失效，列入 R16 的最高優先修復佇列。
Event dispatch is insufficient for some non-standard components. Mitigation: mandated by R17.3; accepted MVP limitation, tracked via R16 when users report specific sites.

**[R-5] 動態載入的表單可能掃描不到 →**
SPA 在使用者點擊後才渲染表單時，掃描會得到 0 個欄位。
緩解：MVP 在掃描結果為 0 時明確回報「此頁面沒有可填寫的欄位」並建議重新整理，不靜默失敗。此情境的完整解法（等待 DOM 穩定或 MutationObserver）列為後續議題。
SPA forms rendered after the click yield 0 fields. Mitigation: an explicit "no fillable fields" message plus a refresh suggestion; a MutationObserver-based solution is a later concern.

**[R-6] 依賴 `el.labels` 擷取問題文字，label 關聯不完整時品質下降 →**
部分網站不用 `<label for>`，改以視覺相鄰的 `<span>`。此時 `questionText` 會退化成父層 `innerText`，可能包含導覽選單等雜訊，降低 LLM 推測準確率。
緩解：已採用 `labels[0]` → 父層 `innerText` → 空字串 的 fallback 鏈並限制長度 100 字；此為推測品質的已知天花板。若實測發現特定網站大量誤判，作為獨立議題處理，不在本變更範圍。
Label association is inconsistent on some sites, degrading `questionText` quality. Mitigation: the fallback chain and 100-character cap already in place; accepted accuracy ceiling for the MVP.

---

## Migration Plan / 遷移計畫

本專案為 greenfield，無既有使用者資料或線上環境，無遷移需求。

啟動步驟 / Startup sequence:

1. `cd backend && npm install`（取得 Express）。
2. 由 `backend/.env.example` 複製為 `backend/.env` 並填入 LiteLLM 連線資訊。
3. `cd backend && npm start` 啟動後端。
4. Chrome 開啟 `chrome://extensions/`，啟用開發人員模式，選擇專案根目錄「載入未封裝項目」。
5. 點擊 Options 圖示，填入後端網址與模型名稱並儲存。
6. 開啟任一含表單的測試頁，點擊工具列圖示並按「分析並填寫」。

回滾策略 / Rollback: 無資料庫與持久化狀態，刪除 `backend/node_modules` 並停用擴充功能即可完全回復。唯一需保留的是 `.env`，其內容由使用者自行保管。

---

## Open Questions / 待確認事項

以下項目已於 `reference/rules_20260724.md` R15 登記，**實作相關 task 前必須取得使用者答覆**（R15：不得自行假設實作）。本節承接 R15 的 Q1–Q6，並標註影響本次變更哪些 task。

The following are registered in R15 of the project rules and **must be answered before implementing the affected tasks** (R15 forbids assuming an implementation). This section carries forward R15's Q1–Q6 and marks which tasks each one blocks.

| # | 問題 / Question | 影響範圍 / Blocks |
|---|---|---|
| Q1 | 敏感欄位判定採「關鍵字清單」還是「ML / 語意分類」？本設計暫定關鍵字清單（D4）。 | `content-script` 敏感欄位 task、`backend/validate.js` |
| Q2 | 擴充功能端維持 vanilla JS，還是導入 TypeScript + Vite？本設計暫定 vanilla JS。 | 影響全部擴充功能檔案的建立方式 |
| Q3 | 後端是否引入 schema 驗證函式庫（zod / ajv）？本設計暫定手動驗證。 | `backend/validate.js`、`backend/server.js` 請求驗證 |
| Q4 | 後端逾時時間具體值為何？失敗時是否降級為本地啟發式猜測？本設計暫定逾時存在但數值待定、無降級。 | `background-worker` 逾時 task、`backend/llm.js` |
| Q5 | 使用者資料是否需要本地持久化以支援「重新填寫」？本設計暫定不持久化。 | `extension-ui` Options 儲存範圍 |
| Q6 | 多頁籤同時觸發如何避免重複呼叫後端？本設計暫定不處理（Non-Goal）。 | 無（Non-Goal，後續變更） |

**另有一項本設計提出、尚未登記於 R15 的問題 / One question raised by this design, not yet registered in R15：**

| # | 問題 / Question | 說明 |
|---|---|---|
| Q7 | 敏感欄位關鍵字清單的初始內容由誰定案？ | D4 決定了「機制」（關鍵字比對）但未決定「清單內容」。清單直接決定 R6.6 的保護範圍，屬產品決策而非技術決策。建議由使用者提供初版清單，或授權依常見身分／財務欄位詞彙擬定後再由使用者覆核。 |
