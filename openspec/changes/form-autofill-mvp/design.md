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
- **不導入 schema 驗證函式庫**（zod / ajv）。已決議（Q3）：契約為「欄位 → 問題 → 答案」的扁平對應，`backend/validate.js` 手寫驗證即可達成同一行為。
  No schema validation library (zod / ajv). Resolved in Q3: the contract is a flat field-to-question-to-answer mapping that hand-written validation in `backend/validate.js` handles.
- **暫不導入 TypeScript 建置流程**，擴充功能端維持 vanilla JS。此為暫定值，Q2 尚未取得使用者答覆。
  No TypeScript build step for now; the extension stays vanilla JS. This is provisional — Q2 is still unanswered.
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

**決策 / Decision**：MVP 以欄位的 `id`、`name`、`questionText` 比對一份關鍵字清單（清單內容見 **D4.1**：憑證、信用卡、銀行帳戶、身分證件、醫療、財務六類），命中即留空並記入 `skipped`。

**理由 / Rationale**：規則式可預測、可測試、無額外相依，符合 MVP 定位。Q1 已決議採關鍵字清單；ML / 語意分類留待有實際誤判率數據後再評估。

**替代方案 / Alternatives considered**：ML / 語意分類 — 覆蓋率可能更好，但引入模型相依且難以測試。完全不保護 — 直接違反 R6.6，本專案最高風險。

**已知不足 / Known limitation**：關鍵字清單會漏掉用詞 unusual 的敏感欄位。緩解見 R3 風險項目。

#### D4.1 — 敏感欄位關鍵字清單初版（Q7 已確認 / Initial sensitive-field keyword list, Q7 resolved）

使用者指示：密碼、信用卡、銀行帳戶這類欄位**一律不填**。以下清單為該指示的具體化。**標示 ⚑ 者為使用者明確指定；標示 ◇ 者為本次設計提出的判斷項目，若您認為過寬或過窄請直接調整。**

The user directed that passwords, credit cards, and bank accounts must never be filled. The list below operationalises that instruction. **⚑ marks items the user named explicitly; ◇ marks judgement calls from this design — trim or extend them as you see fit.**

**比對規則 / Matching rule**（避免誤判的關鍵，務必照做）：

1. 比對對象為欄位的 `id`、`name`、`questionText` 任一者，**任一命中即判定為敏感**。
   Match against a field's `id`, `name`, or `questionText`; a hit on any one of them marks the field sensitive.
2. 比對時**不分大小寫**。
   Matching is case-insensitive.
3. **英文關鍵字採整詞比對**（`\b` 邊界），中文關鍵字採子字串比對。
   **English keywords match on whole words** (`\b` boundaries); CJK keywords match as substrings.
   > 原因：英文若用子字串比對，`pass` 會命中 `passenger`、`pin` 會命中 `shipping`、`cc` 會命中 `account`，造成大量誤判而讓工具失去實用性。中文無詞界概念，只能用子字串。
   > Rationale: substring matching in English makes `pass` hit `passenger`, `pin` hit `shipping`, and `cc` hit `account`. CJK has no word boundaries, so substring is the only option.
4. **只列具體詞組，不列單字**。例如列 `account number` 而不列 `account`，因為 `account` 在一般網站也指「帳號名稱」，那不是敏感資料。
   **List specific phrases, not bare words.** e.g. `account number`, never bare `account`, since a site may use `account` to mean an account name, which is not sensitive.

| 類別 / Category | ⚑/◇ | 關鍵字 / Keywords |
|---|---|---|
| **A. 憑證 / Credentials** | ⚑ | `password`, `passwd`, `pwd`, `new password`, `old password`, `confirm password`, `passcode`, `pin`, `otp`, `security code`, `verification code`, `one-time code`, `密碼`, `確認密碼`, `新密碼`, `舊密碼`, `通行碼`, `驗證碼`, `動態驗證碼`, `簡訊驗證碼`, `安全碼` |
| **B. 信用卡 / Credit card** | ⚑ | `credit card`, `credit card number`, `card number`, `cardholder name`, `cvv`, `cvc`, `csc`, `cid`, `card expiry`, `card expiration`, `信用卡`, `信用卡號`, `卡號`, `持卡人`, `卡片安全碼`, `卡效期`, `卡片有效日期`, `卡片到期日` |
| **C. 銀行帳戶 / Bank account** | ⚑ | `bank account`, `bank account number`, `account number`, `routing number`, `iban`, `swift`, `bic`, `銀行帳號`, `銀行帳戶`, `存款帳號`, `帳戶號碼`, `銀行賬號`, `銀行代碼`, `聯行代號`, `路由號碼` |
| **D. 身分證件 / Identity documents** | ◇ | `national id`, `id number`, `passport number`, `passport`, `driving license`, `driver's license`, `resident certificate`, `tax id`, `taxpayer number`, `身分證`, `身份證`, `身分證號`, `身份證號`, `身分證字號`, `身份證字號`, `身分證號碼`, `身份證號碼`, `護照號碼`, `護照`, `居留證`, `駕照`, `駕照號碼`, `統一編號`, `稅籍` |
| **E. 醫療 / Medical** | ◇ | `medical record`, `medical history`, `health record`, `病歷`, `病歷號`, `就醫紀錄`, `醫療紀錄`, `健康檢查報告`, `過敏史`, `病史`, `健保卡號`, `健保號碼`, `血型` |
| **F. 財務 / Financial** | ◇ | `annual income`, `monthly income`, `net assets`, `savings`, `financial status`, `年收入`, `月收入`, `所得`, `年所得`, `財力`, `財力證明`, `存款`, `資產`, `淨資產`, `薪資`, `財務狀況` |

**刻意不列入者 / Deliberately excluded**（避免把工具變得沒用）：`username` / `使用者名稱` / `account` / `email` / `name` / `address` / `telephone` — 這些雖屬個人資料，但不是憑證或財務憑證，且本工具的用途就是協助填寫一般資料。若您希望連這些也留白，請明確指示後再加入。

**D 與 E、F 為本次設計提出的判斷項目**：使用者指示明確涵蓋 A、B、C 三類。D、E、F 是我依「LLM 絕無可能猜對，且猜錯代價高」的原則延伸。若您認為 D、E、F 會讓填表功能過度受限（例如某些報名表確實需要填年收入），可以要求移除 —— 該決策屬產品範圍，設計層不擅自定案。

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
緩解：MVP 將關鍵字清單設計為易於追加的常數陣列並於 R14 列為變更時必須同步的文件項目；清單缺漏視為已知限制而非缺陷，待實際誤判率數據出現後再決定是否升級為語意分類（Q1 已決議先採關鍵字清單，升級與否屬後續變更）。**本項為 MVP 已知的資料正確性風險，應在 Options 頁告知使用者逐欄檢視。**
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

### 已解決 / Resolved

| # | 問題 / Question | 決議 / Resolution | 日期 |
|---|---|---|---|
| Q1 | 敏感欄位判定採「關鍵字清單」還是「ML / 語意分類」？ | **採關鍵字清單（D4）**。ML / 語意分類留待有實際誤判率數據後再評估。 | 2026-09-27 |
| Q3 | 後端是否引入 schema 驗證函式庫（zod / ajv）？ | **不引入。手動驗證即可。** 使用者理由：本工具是「看到問題才依照問題要回答的格做作答」，請求／回應契約是「欄位 → 問題 → 答案」的扁平對應，不存在需要整份 schema 描述的複雜巢狀結構，因此 schema 驗證函式庫帶來的是額外相依與額外學習成本，換不到對應的價值。 | 2026-09-27 |
| Q7 | 敏感欄位關鍵字清單的初始內容由誰定案？ | **使用者指示涵蓋範圍，設計層擬定具體清單。** 使用者指示：密碼、信用卡、銀行帳戶等類欄位一律不填。設計層依此擬定 A–F 六類共 6 大類關鍵字，其中 A/B/C 為使用者明確指定（⚑），D/E/F 為設計層判斷（◇），待使用者覆核。見 **D4.1**。 | 2026-09-27 |

**Q3 決議的連帶影響 / Consequence of the Q3 decision**：契約的權威定義仍保留在 `backend-api` 規格中（R3 要求跨能力契約須有單一權威定義處），但實作面以 `backend/validate.js` 的手寫函式達成，不引入函式庫。Q3 的理由同時強化了 D6——既然是「逐格作答」，每一格的驗證（選項白名單、`maxLength`、敏感欄位）就必須在該格產生答案的當下完成，這正是 `skipped` 逐項記錄原因碼的設計依據。

The contract's authoritative definition stays in the `backend-api` spec (R3 requires a single authoritative location for cross-capability contracts), but the implementation uses hand-written functions in `backend/validate.js` with no library. The Q3 rationale reinforces D6: given a question-by-question answering model, each field's validation must happen at the moment that field's answer is produced — which is precisely why `skipped` records a reason code per item.

### 仍待確認 / Still open

| # | 問題 / Question | 影響範圍 / Blocks | 暫定值 / Provisional |
|---|---|---|---|
| Q2 | 擴充功能端維持 vanilla JS，還是導入 TypeScript + Vite 建置流程？ | 影響 tasks 1.3 與 3、4、5 全部檔案的建立方式 | vanilla JS（沿用 `auto-fill-html_forms.md` 初版作法）。**此題尚未取得使用者答覆。** |
| Q4 | 後端對 LiteLLM 的逾時秒數為何？失敗時是否降級為本地啟發式猜測？ | `backend-worker` task 4.3、`backend/llm.js` task 2.1 | 逾時存在但數值待定；不做降級（失敗即如實回報，符合 R18.5） |
| Q5 | 使用者資料是否需要本地持久化以支援「重新填寫」？ | `extension-ui` Options 儲存範圍 | 不持久化 |
| Q6 | 多頁籤同時觸發如何避免重複呼叫後端？ | 無（已列為 Non-Goal，後續變更處理） | 不處理 |

**Q2 為目前唯一阻塞實作架構選擇的未決項目**：它決定 1.3、3.1–3.7、4.1–4.4、5.1–5.6 這些 task 要不要建立 `package.json` 與建置設定。Q4、Q5 的暫定值已足以支撐實作，且落入 R18.5 與 R6.7 的禁止行為範圍內，不會產生違規風險。

Q2 is the only remaining open item that blocks an implementation choice: it determines whether tasks 1.3, 3.1–3.7, 4.1–4.4, and 5.1–5.6 need a `package.json` and build config. The provisional values for Q4 and Q5 are sufficient to implement and sit inside the R18.5 / R6.7 prohibitions, so they carry no compliance risk.
