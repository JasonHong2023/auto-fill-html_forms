## Why / 為什麼

通用網頁表單（註冊、報名、求職、申請、結帳）欄位動輒數十個，使用者每填一次都要重複打寫相同或類似的內容，既耗時又容易填錯。本專案提供一個 Chrome 擴充功能，讓使用者在**任意網頁**點擊按鈕，即由 LLM 依欄位語意推測內容並自動填入。

Generic web forms (registration, applications, job applications, checkout) routinely contain dozens of fields, and users repeatedly type the same or similar content — slow and error-prone. This project delivers a Chrome extension that, on **any web page**, infers field content via an LLM and fills it automatically.

此刻是正確時機，因為 `reference/auto-fill-html_forms.md` 的初版程式碼已驗證了核心資料流（掃描 → 呼叫 LLM → 回填）可行，本變更將其工程化為可驗證、可維護、且**不會填錯使用者個資**的 MVP。

Now is the right time because the initial code in `reference/auto-fill-html_forms.md` has already proven the core data flow (scan → call LLM → fill) is viable. This change productizes it into a verifiable, maintainable MVP that **will not corrupt the user's personal data**.

---

## What Changes / 變更內容

本變更建立第一個可運作的端到端 MVP，範圍包含擴充功能四個檔案與後端服務。

This change delivers the first working end-to-end MVP, covering the four extension files plus the backend service.

- **新增 Chrome Extension Manifest V3 骨架**：`manifest.json` 宣告 `activeTab`、`scripting`、`storage` 權限，註冊 `background.js` 為 Service Worker、`popup.html` 為工具列介面，並在 `<all_urls>` 注入 `content.js`。
  Add the Chrome Extension Manifest V3 skeleton: `manifest.json` declaring `activeTab`, `scripting`, `storage`, registering `background.js` as the Service Worker and `popup.html` as the toolbar UI, and injecting `content.js` on `<all_urls>`.

- **新增 Popup 觸發介面與逐欄結果摘要**：`popup.html` / `popup.js` 提供「分析並填寫」按鈕，並依規則 R6.7 顯示「已填入 N 欄／跳過 M 欄」的逐欄結果摘要，包含敏感欄位清單與跳過原因。
  Add the popup trigger UI and a per-field result summary: `popup.html` / `popup.js` provide an "Analyze & Fill" button and, per rule R6.7, display a per-field summary of "N filled / M skipped" including the sensitive-field list and skip reasons.

- **新增 Options 設定頁**：`options.html` / `options.js` 讓使用者設定後端 base URL 與模型名稱，寫入 `chrome.storage.local`。**設定頁不收集任何 API Key**（R6.4）。
  Add an Options page: `options.html` / `options.js` let the user set the backend base URL and model name in `chrome.storage.local`. **The page collects no API key** (R6.4).

- **新增 Content Script 掃描與回填**：`content.js` 掃描 `input` / `textarea` / `select`，為每欄產生穩定識別碼（`id` → `name` → 注入 `data-ai-id`），擷取 label、options 與欄位約束（`required` / `maxlength` / `pattern` / `type`），並在收到答案後依 R17 規則回填：派發 `input` / `change` 事件、選項白名單比對、敏感欄位留空。
  Add Content Script scanning and filling: `content.js` scans `input` / `textarea` / `select`, assigns each field a stable identifier (`id` → `name` → inject `data-ai-id`), extracts labels, options, and constraints (`required` / `maxlength` / `pattern` / `type`), and fills answers per R17 — dispatching `input` / `change` events, validating against the option whitelist, and leaving sensitive fields blank.

- **新增 Service Worker 訊息路由**：`background.js` 接收 `START_FILL`、讀取 `chrome.storage.local` 設定、組裝並送出 `POST {baseUrl}/api/fill`，依 R18.5 設定逾時並將錯誤轉為使用者可見訊息。
  Add the Service Worker message router: `background.js` receives `START_FILL`, reads `chrome.storage.local`, assembles and sends `POST {baseUrl}/api/fill`, sets a timeout per R18.5, and converts errors into user-visible messages.

- **新增後端 API 服務**：`backend/`（`server.js` / `prompt.js` / `llm.js` / `validate.js`）提供 `POST /api/fill`，由環境變數讀取 LiteLLM 金鑰，組裝 Prompt（R18.1 / R18.2 / R18.3），並在回傳前以 options 白名單、`maxlength` 與敏感欄位清單驗證回應。
  Add the backend API service: `backend/` (`server.js` / `prompt.js` / `llm.js` / `validate.js`) exposes `POST /api/fill`, reads the LiteLLM key from environment variables, assembles the prompt (R18.1 / R18.2 / R18.3), and validates the response against the options whitelist, `maxlength`, and the sensitive-field list before returning.

- **明確不做（Non-goals，見 design.md）**：不支援 iframe 內表單、不支援 React 等框架的受控元件深度整合、不主動送出表單、不支援多頁籤並行批次。
  Explicit non-goals (see design.md): no iframe form support, no deep integration with controlled React components, no form auto-submission, no multi-tab batch processing.

---

## Capabilities / 能力

### New Capabilities / 新增能力

- `extension-ui`: Popup 觸發介面、逐欄結果摘要、Options 設定頁與 `chrome.storage.local` 讀寫。
  Popup trigger UI, per-field result summary, Options settings page, and `chrome.storage.local` access.
- `content-script`: 頁面表單欄位掃描、穩定識別碼指派、label / options / 約束擷取，以及依規則回填（含事件派發、選項白名單、敏感欄位留空）。
  Page form field scanning, stable identifier assignment, label / option / constraint extraction, and rule-compliant filling (event dispatch, option whitelist, sensitive fields left blank).
- `background-worker`: Popup 與 Content Script 之間的訊息路由、設定讀取、後端請求組裝、逾時與錯誤可見化。
  Message routing between popup and content script, settings retrieval, backend request assembly, timeout and error visibility.
- `backend-api`: `POST /api/fill` 端點、欄位結構契約（權威定義處）、Prompt 組裝、LiteLLM 呼叫與回應驗證。
  The `POST /api/fill` endpoint, the field-schema contract (authoritative definition), prompt assembly, LiteLLM invocation, and response validation.

### Modified Capabilities / 修改既有能力

無。本專案 `openspec/specs/` 目前為空，本次為首次開案，四个能力皆為新建。
None. `openspec/specs/` is currently empty; this is the project's first change, so all four capabilities are new.

---

## Impact / 影響範圍

**新增檔案 / New files**

```
manifest.json
popup.html            popup.js
options.html          options.js
content.js            background.js
backend/server.js     backend/prompt.js
backend/llm.js        backend/validate.js
backend/package.json  backend/.env.example
```

**新增規格 / New specs**

```
openspec/specs/extension-ui/spec.md
openspec/specs/content-script/spec.md
openspec/specs/background-worker/spec.md
openspec/specs/backend-api/spec.md
```

**相依 / Dependencies**：後端 `express`（版本待確認，R9）。擴充功能端維持零相依。
Backend `express` (version TBD, R9). The extension stays at zero runtime dependencies.

**設定 / Configuration**：後端新增環境變數 `LITELLM_BASE_URL`、`LITELLM_API_KEY`、`LITELLM_MODEL`、`BACKEND_PORT`。`.env` 已由 `.gitignore` 排除。
The backend adds environment variables `LITELLM_BASE_URL`, `LITELLM_API_KEY`, `LITELLM_MODEL`, `BACKEND_PORT`. `.env` is already excluded by `.gitignore`.

**安全影響 / Security impact**：頁面欄位文字將離開瀏覽器、送往本機後端與 LiteLLM。此為資料流向變更，必須在 design.md 的 Risks 段落與使用者在 Options 頁明確揭露（R18.4）。
Field text leaves the browser and is sent to the local backend and LiteLLM. This is a data-flow change and must be disclosed in design.md's Risks section and to the user in the Options page (R18.4).

**不受影響 / Not affected**：既有 `reference/` 文件（`auto-fill-html_forms.md` 為參考來源，`rules_20260724.md` 已於 v1.1 完成轉向）。
Existing `reference/` documents (`auto-fill-html_forms.md` is the reference source; `rules_20260724.md` was already pivoted in v1.1).
