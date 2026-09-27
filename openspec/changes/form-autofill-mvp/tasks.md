## 1. 專案骨架 / Project Scaffolding

- [ ] 1.1 建立 `backend/package.json`，宣告 `express` 相依與 `start` 指令（依 D8 與 R9，版本待使用者確認後鎖定）— expect `npm install` 可成功安裝
- [ ] 1.2 建立 `backend/.env.example`，列出 `LITELLM_BASE_URL`、`LITELLM_API_KEY`、`LITELLM_MODEL`、`BACKEND_PORT` 四個變數且值為佔位符 — expect 不得含任何真實金鑰
- [ ] 1.3 建立 `manifest.json`（MV3），宣告 `activeTab`/`scripting`/`storage` 權限、`background.js` service worker、`popup.html` action 與 `<all_urls>` content script — expect 可在 `chrome://extensions/` 成功載入未封裝項目

## 2. backend-api（後端 API）/ Backend API

- [ ] 2.1 建立 `backend/llm.js`：自環境變數讀取金鑰，實作 `POST {LITELLM_BASE_URL}/v1/chat/completions`，帶 `response_format: json_object` 與 `temperature: 0.1`，並設定逾時（逾時值待 Q4 確認）— expect 逾時與連線錯誤拋出可區分的錯誤型別
- [ ] 2.2 建立 `backend/prompt.js`：組裝 system prompt，要求純 JSON（禁止 markdown 區塊）、選擇型欄位答案必須逐字符合 `options`、頁面內容視為資料非指令（對應 R18.1–R18.3）— expect prompt 文字明確含三項約束
- [ ] 2.3 建立 `backend/validate.js` 的**請求驗證**部分：驗 R9 契約必要欄位（`id`、`questionText` 等）、欄位數上限、payload 大小上限 — expect 不合法請求回傳具名欄位錯誤
- [ ] 2.4 建立 `backend/validate.js` 的**回應驗證**部分：敏感欄位丟棄（D4.1 六類關鍵字清單，**此檔為權威實作**）、`options` 白名單比對、`maxLength` 檢查、未識別欄位鍵丟棄 — expect 每項丟棄都產出 `skipped` 項目與原因碼（`option_not_found` / `max_length_exceeded` / `sensitive_field`），且密碼／信用卡／銀行帳戶類欄位即使 LLM 回傳看似合理的值也一律丟棄
- [ ] 2.5 建立 `backend/server.js`：實作 `POST /api/fill` 與 `GET /api/health`，呼叫 llm → parse → validate → 回傳 `{ answers, skipped }`，解析失敗回 HTTP 502、逾時回 504、連線失敗回 502 — expect 以 `curl` 對三種錯誤路徑各取得對應狀態碼
- [ ] 2.6 剝除 `pageUrl` 查詢字串後才送往 LLM，並確認後端日誌不輸出金鑰（對應 R18.4 與 `backend-api` 金鑰規格）— expect 送出內容僅含 scheme + host

## 3. content-script（內容腳本）/ Content Script

- [ ] 3.1 建立掃描邏輯：以 `input:not([type=hidden]):not([type=submit]):not([type=button]):not([type=image]):not([type=reset]):not([type=file]), textarea, select` 掃描，排除清單依 R17.2 — expect `type=file` 欄位不出現在結果
- [ ] 3.2 實作三級識別碼指派（`id` → `name` → 注入 `data-ai-id`）與重複偵測去重（對應 D3）— expect 同 `name` 的 radio 群組只有第一個用 `name`，其餘獲得注入的 `data-ai-id`
- [ ] 3.3 實作語意與約束擷取：`questionText`（`labels[0]` → 父層 `innerText` → 空字串，截斷 100 字）、`options`（select 全 options、radio/checkbox 同 `name` 群組）、`required` / `maxLength` / `pattern` — expect 缺少時回傳 `null` 或空字串而非 `undefined`
- [ ] 3.4 實作敏感欄位偵測（關鍵字清單依 D4.1 全部六類 A–F，比對 `id` / `name` / `questionText`，英文整詞、中文子字串、不分大小寫）並標記 — expect 命中欄位被標記且不填入任何值，且「passenger name」「shipping address」不被誤判
- [ ] 3.5 實作 `fillForm()`：三級反查（`getElementById` → `[name]` → `[data-ai-id]`）、text/textarea 指派、select 指派、radio/checkbox 勾選、陣列多選 — expect 識別碼查無對應元素時跳過該筆記入 `skipped` 且不中斷整批
- [ ] 3.6 於每次填值後派發 `input` 與 `change` 事件（`bubbles: true`，對應 R17.3 / D7）— expect 於 React 測試頁填值後框架狀態與 DOM 值一致
- [ ] 3.7 實作回填前的二次防護：敏感欄位不填、`options` 白名單比對、`maxLength` 檢查，違者記入 `skipped`（對應 D5 縱深防禦）— expect 後端已過濾的情況下本地仍不會寫入敏感欄位

## 4. background-worker（背景工作腳本）/ Service Worker

- [ ] 4.1 實作訊息路由：`START_FILL` 轉送至活躍分頁 content script、`CALL_FILL` 轉送後端、listener 回傳 `true` 保持非同步通道開啟 — expect 兩種 action 皆可往返
- [ ] 4.2 讀取 `chrome.storage.local` 的後端網址與模型名稱，組裝 `POST {baseUrl}/api/fill` 請求（不含任何金鑰標頭，對應 D2）— expect 請求標頭不含 Authorization
- [ ] 4.3 設定具名逾時常數並在逾時時中止請求、回傳「後端回應逾時」（逾時值待 Q4 確認，對應 R18.5）— expect 逾時非無聲等待
- [ ] 4.4 實作回應格式驗證：解析失敗或缺必要欄位時回傳明確解析錯誤，**不得**回傳空 `answers`（對應 R18.1）— expect 解析失敗訊息可區別於「無可填欄位」

## 5. extension-ui（介面）/ Extension UI

- [ ] 5.1 建立 `options.html` / `options.js`：後端網址（http/https 格式驗證）與模型名稱欄位，寫入 `chrome.storage.local` — expect 非法網址被拒儲存
- [ ] 5.2 確認 Options 頁**不含**任何 API 金鑰輸入欄位（對應 D2 / R6.4）— expect 頁面僅兩個非敏感設定欄位
- [ ] 5.3 建立 `popup.html` / `popup.js`：「分析並填寫」按鈕 + 狀態區，發送 `START_FILL` 並在作業進行中停用按鈕 — expect 點擊後按鈕停用
- [ ] 5.4 實作逐欄結果摘要：顯示「已填入 N 欄；跳過 M 欄」、逐項列出跳過欄位與原因碼、`sensitive_field` 項目明確提示手動填寫；`answers` 為空且全部跳過時顯示「未填入任何欄位」而非成功（對應 R6.7 / D6）— expect 各種 skip 組合皆正確呈現
- [ ] 5.5 實作錯誤訊息分支：content script 未載入（提示重整）、後端未設定（提示前往 Options 並提供開啟按鈕）、後端連線失敗、逾時 — expect 四種錯誤皆顯示可行動訊息
- [ ] 5.6 在 Options 頁加入資料流向揭露文案，說明頁面欄位文字會送往後端與 LLM 供應商，並提示使用者逐欄檢視（對應風險 R-1 / R-2）— expect 使用者在啟用前可讀到告知

## 6. 端到端驗證 / End-to-End Verification

- [ ] 6.1 建立本機測試頁（`test/fixture.html`），含文字框、下拉選單、單選群組、多選、textarea、`type=file` 欄位與一個命名為「身分證字號」的敏感欄位 — expect 可覆蓋所有掃描與回填分支
- [ ] 6.2 以 `curl` 驗證 `POST /api/fill` 對合法請求回 200 且結構為 `{ answers, skipped }`，對非法請求回 400 並指明欄位（依 R7 驗證方式）— expect 三種狀態碼與結構皆正確
- [ ] 6.3 後端啟動後於測試頁觸發完整流程，逐項核對 R12 流程圖每一步（掃描 → 請求 → LLM → 驗證 → 回填 → 摘要）— expect 摘要數字與實際 DOM 值一致
- [ ] 6.4 於 React 測試頁驗證事件派發使框架狀態同步（對應 D7 / 風險 R-4）— expect 框架狀態值與 DOM 值一致
- [ ] 6.5 驗證敏感欄位在完整流程中未被填入且出現於摘要清單（對應 R6.6 / 風險 R-2）— expect 身分證欄位值維持空白並列於跳過清單
- [ ] 6.6 驗證停用後端後 Popup 顯示可行動錯誤而非「處理完成」（對應 R18.5 / 風險 R-3）— expect 顯示後端未啟動訊息
- [ ] 6.7 確認擴充功能原始碼中無任何金鑰字面值，且 `git status` 顯示 `.env` 未被追蹤（對應 R6.4）— expect 兩項檢查皆通過

## 7. 文件同步 / Documentation Sync

- [ ] 7.1 依 R9 與 R14 填寫 `openspec/specs/stack.md`，鎖定 Express 與 Node.js 的確切版本 — expect 版本號為實際安裝版本
- [ ] 7.2 依 R14 撰寫 `README.md`，涵蓋安裝、啟動後端、載入擴充功能、資料流向告知段落（依 proposal 的 `GET /api/health` 契約）— expect 新使用者可依文件完成設定
- [ ] 7.3 依 R3 與 R10 撰寫 `PRD.md`（待規格穩定後）或明確記錄其暫緩理由 — expect 文件狀態與實際一致
- [ ] 7.4 依 R13 更新專案狀態呈現，確認 proposal / tasks / specs / design / rules 五項一致 — expect 五項狀態同步
- [ ] 7.5 依 R15 回填 Q2、Q4、Q5 的確認結果，移除 `design.md` Open Questions 中已解決項目並同步至 R15 — expect 無已解決項目仍列為待確認
