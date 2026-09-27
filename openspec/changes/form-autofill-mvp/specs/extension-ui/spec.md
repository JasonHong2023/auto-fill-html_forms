## ADDED Requirements

### Requirement: 使用者可從 Popup 觸發表單填寫
The system SHALL provide an "分析並填寫 / Analyze & Fill" button in the popup that asks the active tab's content script to begin form filling, and SHALL disable that button while a fill operation is in progress.

#### Scenario: 點擊按鈕觸發填寫
- **WHEN** 使用者點擊 Popup 的「分析並填寫」按鈕，且目標分頁已載入 content script
- **THEN** 系統將 `START_FILL` 訊息傳送至該分頁的 content script，並停用按鈕直到收到結果或錯誤
- **WHEN** the user clicks the "Analyze & Fill" button and the target tab has the content script loaded
- **THEN** the system sends a `START_FILL` message to that tab's content script and disables the button until a result or error arrives

#### Scenario: 目標分頁尚未載入 content script
- **WHEN** content script 尚未注入該分頁（例如該頁面在擴充功能安裝前就已開啟）
- **THEN** Popup 顯示「請重新整理頁面後再試 / Please refresh the page and try again」，且不送出任何後端請求
- **WHEN** the content script has not been injected into the target tab
- **THEN** the popup shows "Please refresh the page and try again" and sends no backend request

### Requirement: Popup 顯示逐欄結果摘要
The popup SHALL display a per-field result summary after every fill attempt, stating how many fields were filled, how many were skipped, and the reason for each skip. The system MUST NOT display a bare success message that omits skips.

#### Scenario: 顯示已填與跳過統計
- **WHEN** 後端回傳 `answers` 含有 8 個欄位且 `skipped` 含有 3 個欄位
- **THEN** Popup 顯示「已填入 8 欄；跳過 3 欄」並逐項列出 3 個跳過欄位的名稱與原因
- **WHEN** the backend returns `answers` with 8 fields and `skipped` with 3 fields
- **THEN** the popup shows "8 fields filled; 3 skipped" and lists each skipped field's name and reason

#### Scenario: 全部欄位皆被跳過
- **WHEN** `answers` 為空物件且 `skipped` 含有全部掃描到的欄位
- **THEN** Popup 顯示「未填入任何欄位」並列出所有跳過原因，且不得顯示為成功完成
- **WHEN** `answers` is an empty object and `skipped` contains every scanned field
- **THEN** the popup shows "No fields were filled" with all skip reasons, and MUST NOT present this as a success

#### Scenario: 顯示敏感欄位提示
- **WHEN** `skipped` 中存在 `reason` 為 `sensitive_field` 的項目
- **THEN** Popup 逐項列出該欄位名稱並提示「請手動填寫 / Please fill this in manually」
- **WHEN** `skipped` contains an entry whose `reason` is `sensitive_field`
- **THEN** the popup lists that field by name and prompts "Please fill this in manually"

### Requirement: 填寫失敗時顯示可理解的錯誤訊息
The popup SHALL surface backend and network failures as an explicit error message, and MUST NOT display a success or "process finished" state when the operation failed.

#### Scenario: 後端未啟動
- **WHEN** Service Worker 收到網路錯誤（後端未啟動或連線被拒）
- **THEN** Popup 顯示「無法連線至後端服務，請確認後端已啟動 / Cannot reach the backend service, please check that it is running」
- **WHEN** the service worker receives a network error
- **THEN** the popup shows "Cannot reach the backend service, please check that it is running"

#### Scenario: 後端回應非 2xx
- **WHEN** 後端回應 HTTP 400 且訊息指出欄位結構無效
- **THEN** Popup 顯示該錯誤訊息並結束本次填寫流程，不顯示任何填入結果
- **WHEN** the backend responds with HTTP 400 indicating an invalid field structure
- **THEN** the popup shows that error and ends the fill operation without reporting any filled fields

### Requirement: 使用者可在 Options 頁設定後端連線資訊
The system SHALL provide an Options page where the user can set the backend base URL and the model name, persisting them in `chrome.storage.local`.

#### Scenario: 儲存後端設定
- **WHEN** 使用者在 Options 頁輸入後端網址與模型名稱並儲存
- **THEN** 設定寫入 `chrome.storage.local`，且 Options 頁顯示儲存成功的確認訊息
- **WHEN** the user enters a backend URL and model name in the Options page and saves
- **THEN** the settings are written to `chrome.storage.local` and the page confirms the save

#### Scenario: 驗證後端網址格式
- **WHEN** 使用者輸入的後端網址不是合法的 http 或 https URL
- **THEN** Options 頁顯示格式錯誤訊息並拒絕儲存該值
- **WHEN** the user enters a backend URL that is not a valid http or https URL
- **THEN** the page shows a format error and refuses to save the value

### Requirement: 設定頁不收集任何 API 金鑰
The Options page MUST NOT collect, store, or transmit any API key, token, or secret. The LiteLLM credential SHALL exist only in the backend's environment variables.

#### Scenario: 設定頁無金鑰欄位
- **WHEN** 使用者開啟 Options 頁
- **THEN** 頁面僅呈現後端網址與模型名稱欄位，不存在任何密鑰輸入欄位
- **WHEN** the user opens the Options page
- **THEN** the page presents only the backend URL and model name fields, with no credential input of any kind

#### Scenario: 擴充功能不持有 LiteLLM 金鑰
- **WHEN** 檢視擴充功能的所有原始碼檔案
- **THEN** 檔案中不存在任何 LiteLLM API 金鑰字面值
- **WHEN** all extension source files are inspected
- **THEN** no file contains a literal LiteLLM API key

### Requirement: 設定未完成時引導使用者前往設定頁
The system SHALL detect a missing backend base URL and SHALL direct the user to the Options page instead of attempting a fill operation.

#### Scenario: 缺少後端網址時阻擋填寫
- **WHEN** 使用者點擊「分析並填寫」但 `chrome.storage.local` 中不存在後端網址
- **THEN** Popup 顯示「尚未設定後端連線 / Backend connection is not configured」並提供開啟 Options 頁的按鈕，且不掃描頁面也不呼叫後端
- **WHEN** the user clicks "Analyze & Fill" but no backend URL exists in `chrome.storage.local`
- **THEN** the popup shows "Backend connection is not configured" with a button to open the Options page, and neither scans the page nor calls the backend
