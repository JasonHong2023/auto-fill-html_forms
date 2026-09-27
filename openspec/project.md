# Project Context

> 專案：通用 HTML 表單自動填寫外掛 / Project: Generic HTML Form Auto-Filler Extension
> 規則文件 / Rules: [`reference/rules_20260724.md`](../reference/rules_20260724.md)（v1.2）
> 本文件為 OpenSpec 的專案技術棧與慣例來源，內容須與 R9（技術棧鎖定）保持一致。
> This file is the source of truth for OpenSpec project stack and conventions,
> and must stay consistent with R9 (Tech Stack Lock) in the project rules.

---

## Purpose / 專案目的

一個 Chrome Extension（Manifest V3），讓使用者在**任意網頁**點擊按鈕後，由 LLM 推測並自動填寫表單欄位（文字框、單選、多選、下拉選單、textarea）。

A Chrome Extension (Manifest V3) that lets a user click a button on **any web page** and have an LLM infer and auto-fill form fields (text inputs, radios, checkboxes, selects, textareas).

本專案會填寫使用者的**真實個人資料**，因此失敗路徑必須可見、猜測內容必須可覆寫——這是本專案的最高風險基準。
This project writes the user's **real personal data**, so every failure path must be visible and every inferred value must be overwritable. This is the project's governing risk baseline.

---

## Tech Stack / 技術棧

### Extension（擴充功能端）

| 項目 / Item | 選擇 / Choice |
|---|---|
| Manifest | Chrome **Manifest V3**（不可降級） |
| 語言 / Language | Vanilla JavaScript（無框架、**無建置步驟**）— `manifest.json` 置於專案根目錄，「載入未封裝項目」即可執行（Q2 已確認） |
| 權限 / Permissions | `activeTab`、`scripting`、`storage` |
| 設定儲存 / Settings | `chrome.storage.local`，**僅存非敏感設定**（後端 base URL、模型名稱） |
| 入口 / Entry points | `popup.html` / `options.html`（設定頁） |

### Backend（後端）

| 項目 / Item | 選擇 / Choice |
|---|---|
| Runtime | Node.js（版本待確認） |
| Framework | Express（版本待確認） |
| LLM | OpenAI-compatible endpoint（LiteLLM `/v1/chat/completions`），`temperature: 0.1`，`response_format: json_object` |
| 金鑰 / Secrets | 僅存在後端環境變數（`.env`），**絕不進 Git、絕不進擴充功能** |

**後端環境變數 / Backend env vars**：

```
LITELLM_BASE_URL   # 例如 http://localhost:4000
LITELLM_API_KEY    # 僅後端持有
LITELLM_MODEL      # 例如 gemini-2.0-flash
BACKEND_PORT       # 部署環境決定
```

---

## Capabilities / 能力劃分

| 能力 / Capability | 職責 / Responsibility | 主要檔案 / Key files |
|---|---|---|
| `extension-ui` | Popup 觸發介面、Options 設定頁、`chrome.storage.local` 讀寫 | `popup.html` `popup.js` `options.html` `options.js` |
| `content-script` | 掃描頁面欄位、擷取 label/options/約束、執行回填 | `content.js` |
| `background-worker` | 訊息路由、讀取設定、組裝後端請求、逾時與錯誤處理 | `background.js` `manifest.json` |
| `backend-api` | HTTP 端點、Prompt 組裝、呼叫 LiteLLM、回應驗證 | `backend/server.js` `backend/prompt.js` `backend/llm.js` `backend/validate.js` |

**跨能力契約 / Cross-capability contract**：`content-script` 送出的欄位結構與 `backend-api` 的請求／回應格式，權威定義位於 `openspec/specs/backend-api/spec.md`；`content-script` 以 reference 方式引用，不得各自定義一份。權威定義 / The field schema contract is authoritative in `openspec/specs/backend-api/spec.md`; `content-script` references it rather than redefining it.

---

## Project Structure / 專案結構

```
auto-fill-html_forms/
├── manifest.json              # MV3 設定檔
├── popup.html / popup.js      # 工具列 Popup：觸發與結果摘要
├── options.html / options.js  # 設定頁：後端網址、模型名稱
├── content.js                 # DOM 掃描與回填
├── background.js              # Service Worker：訊息路由與後端請求
├── backend/
│   ├── server.js              # Express 路由與啟動
│   ├── prompt.js              # Prompt 模板組裝
│   ├── llm.js                 # LiteLLM 客戶端
│   └── validate.js            # 回應驗證（options 白名單、maxLength、敏感欄位）
├── openspec/                  # OpenSpec 規格
└── reference/                 # 規則與參考程式碼
```

---

## Conventions / 慣例

1. **雙語書寫 / Bilingual writing**：所有程式碼註解與 `openspec/changes/**` 文件以繁體中文 + 英文逐句並列書寫（R1）。
2. **需求措辭 / Requirement wording**：規格需求使用 `SHALL` / `MUST`（RFC 2119）。
3. **禁止清單優先 / Anti-patterns first**：動工前先讀 R6（10 條禁止行為）、R17（DOM 正確性 5 點）、R18（LLM 契約 5 點）。
4. **秘密管理 / Secret handling**：任何 `.env` 不得 commit（`.gitignore` 已含 `.env` 與 `.env.*`）。
5. **不可主動送出 / Never auto-submit**：工具**永不**觸發 `form.submit()` 或點擊送出鈕（R6.7）。
6. **失敗必須可見 / Failures must be visible**：解析失敗、逾時、選項不符一律記入 `skipped` 並在 Popup 摘要顯示，不得靜默回傳空結果（R6.1、R18.1、R18.5）。

---

## Testing / 測試

- **擴充功能端**：以 Chrome DevTools 檢查 Service Worker 與 Content Script Console；以人工測試頁驗證掃描與回填（R7 範例）。
- **後端端點**：以 `curl` 直接打 `POST /api/fill` 驗證請求驗證、Prompt 組裝與回應驗證。
- **跨端整合**：後端啟動後於瀏覽器端觸發一次完整流程，核對 R12 流程圖的每一步。

---

## Commands / 指令

```bash
# 安裝 / Install
cd backend && npm install

# 啟動後端（需先備好 .env）/ Run backend
cd backend && npm start

# 載入擴充功能 / Load extension
# chrome://extensions/ → 開啟開發人員模式 → 載入未封裝項目 → 選專案根目錄

# OpenSpec
openspec list
openspec status --change <name> --json
openspec validate <name> --strict
```

---

## Open Questions / 待確認事項

詳見 `reference/rules_20260724.md` R15。已解決：Q1（採關鍵字清單）、Q2（維持 vanilla JS，無建置流程，「載入未封裝項目」即可執行）、Q3（不引入 schema 驗證函式庫）、Q7（關鍵字清單初版，見 design.md D4.1）。仍待確認：Q4（逾時秒數與降級）、Q5（資料持久化）、Q6（多頁籤併發，已列 Non-Goal）。**目前沒有阻塞實作的未決事項**；Q4 與 Q5 的暫定值已足以支撐實作且不會違反 R6.7 與 R18.5。
See R15 of the project rules. Resolved: Q1 (keyword list), Q2 (vanilla JS, no build step, runs from "Load unpacked"), Q3 (no schema validation library), Q7 (initial keyword list; see design.md D4.1). Still open: Q4 (timeout seconds and degradation), Q5 (data persistence), Q6 (multi-tab concurrency; already a Non-Goal). **No open item blocks implementation**; the provisional values for Q4 and Q5 suffice and violate neither R6.7 nor R18.5.

---

## Conventions 不變式 / Invariants

以下為跨變更持續成立的不變式，任何變更若違反其中一項，必須在 `design.md` 中明確說明理由並取得使用者確認。
The following invariants hold across all changes. Any change violating one must justify it in `design.md` and obtain explicit user confirmation.

1. LiteLLM API Key 只存在於後端環境變數，永不進入擴充功能程式碼或 Git。
2. 擴充功能永不主動送出表單。
3. 敏感欄位（身分／財務類）永不填入 LLM 生成值。
4. 所有失敗路徑都必須在使用者可見的摘要中呈現。
5. 頁面內容一律視為不可信資料，不得提升為 LLM 指令層級（R18.3）。
6. 送給 LiteLLM 的內容僅限完成填寫所必需的欄位結構描述（R18.4）。
