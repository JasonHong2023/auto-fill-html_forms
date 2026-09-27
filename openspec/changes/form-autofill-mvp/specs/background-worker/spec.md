## ADDED Requirements

### Requirement: 路由 Popup 與 Content Script 的訊息
The service worker SHALL relay a `START_FILL` action from the popup to the content script of the active tab, and SHALL relay the content script's `CALL_FILL` action containing the scanned field structure to the backend. It SHALL return a `true` return value from the message listener to keep the asynchronous channel open.

#### Scenario: 轉送 START_FILL 訊息
- **WHEN** Popup 發送 `{ action: "START_FILL" }` 給 service worker
- **THEN** service worker 取得當前活躍分頁 ID，並將 `START_FILL` 轉送至該分頁的 content script
- **WHEN** the popup sends `{ action: "START_FILL" }` to the service worker
- **THEN** the service worker resolves the active tab id and relays `START_FILL` to that tab's content script

#### Scenario: 目標分頁無法接收訊息
- **WHEN** `chrome.tabs.sendMessage` 因該分頁未載入 content script 而失敗
- **THEN** service worker 回傳明確的錯誤狀態，Popup 据此顯示「請重新整理頁面後再試」，且不記錄未捕捉的例外
- **WHEN** `chrome.tabs.sendMessage` fails because the tab has no content script
- **THEN** the service worker returns an explicit error status so the popup can show "Please refresh the page and try again", and does not leave an uncaught exception

#### Scenario: 非預期 action 不被處理
- **WHEN** service worker 收到未辨識的 action
- **THEN** 該訊息被忽略且不產生任何後端請求
- **WHEN** the service worker receives an unrecognised action
- **THEN** the message is ignored and no backend request is made

### Requirement: 讀取設定並組裝後端請求
The service worker SHALL read the backend base URL and model name from `chrome.storage.local` and SHALL construct a `POST` request to `{baseUrl}/api/fill` whose body matches the field structure contract defined in `backend-api`.

#### Scenario: 組裝請求
- **WHEN** `chrome.storage.local` 中存在後端網址 `http://localhost:3000` 與模型名稱 `gemini-2.0-flash`，且 content script 送來 5 個欄位
- **THEN** service worker 送出 `POST http://localhost:3000/api/fill`，主體包含 `pageUrl` 與 5 個欄位的 `fields` 陣列，並帶有 `Content-Type: application/json`
- **WHEN** `chrome.storage.local` holds the backend URL `http://localhost:3000` and model `gemini-2.0-flash`, and the content script supplies 5 fields
- **THEN** the service worker sends `POST http://localhost:3000/api/fill` with a body containing `pageUrl` and a `fields` array of 5 fields, and the `Content-Type: application/json` header

#### Scenario: 設定缺失時中止
- **WHEN** `chrome.storage.local` 中不存在後端網址
- **THEN** service worker 不發出任何網路請求，回傳「尚未設定後端連線」的錯誤狀態
- **WHEN** no backend URL exists in `chrome.storage.local`
- **THEN** the service worker makes no network request and returns a "backend connection is not configured" error status

#### Scenario: 後端請求不攜帶任何金鑰
- **WHEN** service worker 組裝並送出後端請求
- **THEN** 請求標頭中不包含任何 API 金鑰或 Authorization 標頭
- **WHEN** the service worker assembles and sends the backend request
- **THEN** the request headers contain no API key and no Authorization header

### Requirement: 對後端請求設定逾時
The service worker SHALL apply a timeout to the backend request and SHALL surface a timeout as a distinct, user-visible error rather than leaving the operation pending indefinitely.

#### Scenario: 後端逾時
- **WHEN** 後端在設定的逾時時間內未回應
- **THEN** service worker 中止該請求並回傳「後端回應逾時」錯誤，Popup 顯示此訊息
- **WHEN** the backend does not respond within the configured timeout
- **THEN** the service worker aborts the request and returns a "backend response timed out" error that the popup displays

#### Scenario: 逾時值可辨識
- **WHEN** 檢查 service worker 的逾時設定
- **THEN** 逾時毫秒數為明確的具名常數或設定項，而非散落於程式碼中的魔術數字
- **WHEN** the service worker's timeout setting is inspected
- **THEN** the timeout in milliseconds is an explicit named constant or setting rather than a magic number scattered in code

### Requirement: 驗證後端回應格式
The service worker SHALL validate the backend response before forwarding it to the content script, and MUST NOT forward a malformed response. On a validation failure it SHALL report a parse error rather than an empty result set.

#### Scenario: 接受格式正確的回應
- **WHEN** 後端回應包含 `answers` 物件與 `skipped` 陣列
- **THEN** service worker 將完整回應轉送給 content script 進行回填
- **WHEN** the backend response contains an `answers` object and a `skipped` array
- **THEN** the service worker forwards the full response to the content script for filling

#### Scenario: 拒絕格式錯誤的回應
- **WHEN** 後端回應的 JSON 無法解析或缺少必要欄位
- **THEN** service worker 不轉送給 content script，並回傳明確的解析錯誤訊息
- **WHEN** the backend response JSON cannot be parsed or is missing required fields
- **THEN** the service worker does not forward it to the content script and returns an explicit parse-error message

#### Scenario: 解析失敗不得靜默
- **WHEN** 回應解析失敗
- **THEN** 回報的訊息 MUST 指明為解析錯誤，MUST NOT 呈現為「沒有可填寫的欄位」或「處理完成」
- **WHEN** response parsing fails
- **THEN** the reported message MUST identify a parse error and MUST NOT appear as "no fillable fields" or "process finished"
