## ADDED Requirements

### Requirement: 掃描頁面可填寫欄位
The content script SHALL scan the current page for fillable fields using a selector covering `input`, `textarea`, and `select`, and MUST exclude `input` elements whose `type` is `hidden`, `submit`, `button`, `image`, `reset`, or `file`.

#### Scenario: 掃描各類欄位
- **WHEN** 頁面同時包含文字框、單選、多選、下拉選單與 textarea
- **THEN** 全部欄位都被納入掃描結果，且各自帶有符合其類型的 `type` 與 `tag`
- **WHEN** the page contains text inputs, radios, checkboxes, selects, and textareas
- **THEN** all fields are included in the scan result, each carrying the `type` and `tag` matching its kind

#### Scenario: 排除非輸入欄位
- **WHEN** 頁面包含 `type` 為 `hidden`、`submit`、`button`、`image`、`reset` 或 `file` 的 input
- **THEN** 這些欄位不出現在掃描結果中
- **WHEN** the page contains inputs whose `type` is `hidden`, `submit`, `button`, `image`, `reset`, or `file`
- **THEN** those fields do not appear in the scan result

#### Scenario: 頁面無可填欄位
- **WHEN** 掃描結果為 0 個欄位
- **THEN** content script 立即回報「此頁面沒有可填寫的欄位」且不呼叫後端 API
- **WHEN** the scan yields 0 fields
- **THEN** the content script reports "this page has no fillable fields" and does not call the backend API

### Requirement: 為每個欄位指派穩定識別碼
The content script SHALL assign every scanned field a non-empty identifier that is unique within that scan, using the existing `id`, then the existing `name`, then injecting `data-ai-id="ai_input_<index>"` into the element.

#### Scenario: 優先採用既有 id
- **WHEN** 欄位已存在 `id` 屬性
- **THEN** 識別碼採用該 `id`，且不修改元素的 `id` 屬性
- **WHEN** a field already has an `id` attribute
- **THEN** the identifier is that `id`, and the element's `id` attribute is left unchanged

#### Scenario: 缺少 id 時注入 data-ai-id
- **WHEN** 欄位沒有 `id` 但有 `name`
- **THEN** 識別碼採用該 `name`，且不注入 `data-ai-id`
- **WHEN** a field has no `id` but has a `name`
- **THEN** the identifier is that `name`, and no `data-ai-id` is injected

#### Scenario: id 與 name 皆缺少
- **WHEN** 欄位既沒有 `id` 也沒有 `name`
- **THEN** content script 將 `data-ai-id="ai_input_<index>"` 注入該元素，並以該值為識別碼
- **WHEN** a field has neither `id` nor `name`
- **THEN** the content script injects `data-ai-id="ai_input_<index>"` into the element and uses it as the identifier

#### Scenario: 識別碼不得重複
- **WHEN** 頁面上存在兩個 `name` 相同的欄位（例如同名 radio 群組）
- **THEN** 同群組內僅第一個欄位以該 `name` 為識別碼，其餘欄位改用注入的 `data-ai-id`，確保識別碼不重複
- **WHEN** the page has two fields sharing the same `name` (e.g. a radio group)
- **THEN** only the first uses that `name` as its identifier; the rest receive an injected `data-ai-id`, keeping identifiers unique

### Requirement: 擷取欄位語意資訊與約束
The content script SHALL extract, for each scanned field, a `questionText` derived from the field's label, the available `options` for choice-type fields, and the constraints `required`, `maxLength`, `pattern`, and `type`.

#### Scenario: 從 label 擷取問題文字
- **WHEN** 欄位具有關聯的 `<label>` 元素
- **THEN** `questionText` 取自第一個關聯 label 的文字，並截斷至 100 字元
- **WHEN** a field has an associated `<label>` element
- **THEN** `questionText` comes from the first associated label's text, truncated to 100 characters

#### Scenario: 無 label 時改用父層文字
- **WHEN** 欄位沒有關聯的 `<label>` 元素
- **THEN** `questionText` 取自其父層元素的文字；若父層亦無文字，則回傳空字串而非 `undefined`
- **WHEN** a field has no associated `<label>` element
- **THEN** `questionText` comes from its parent element's text, or an empty string rather than `undefined` if the parent has none

#### Scenario: 收集下拉選單選項
- **WHEN** 欄位為 `select`
- **THEN** `options` 為其所有 `<option>` 的值陣列；若無 option 則為 `null`
- **WHEN** a field is a `select`
- **THEN** `options` is an array of all its `<option>` values, or `null` if it has none

#### Scenario: 收集單選與多選選項
- **WHEN** 欄位為 `radio` 或 `checkbox`
- **THEN** `options` 陣列收集同一 `name` 群組內所有成員的 `value`，而非僅該元素自身的值
- **WHEN** a field is a `radio` or `checkbox`
- **THEN** the `options` array collects the `value` of every member sharing the same `name` group, not only that element's own value

#### Scenario: 擷取欄位約束
- **WHEN** 欄位帶有 `required`、`maxlength` 或 `pattern` 屬性
- **THEN** 對應欄位分別回傳 `required`、`maxLength`、`pattern`，缺少時回傳 `null`
- **WHEN** a field carries `required`, `maxlength`, or `pattern` attributes
- **THEN** the corresponding `required`, `maxLength`, `pattern` fields reflect them, or `null` when absent

### Requirement: 依穩定識別碼回填欄位值
The content script SHALL resolve each answer's target element by identifier using the order `getElementById`, then `[name="..."]`, then `[data-ai-id="..."]`, and MUST skip any answer whose identifier resolves to no element.

#### Scenario: 以識別碼解析元素
- **WHEN** 答案的識別碼對應一個帶有該 `id` 的元素
- **THEN** 該元素被填入答案值
- **WHEN** an answer's identifier matches an element with that `id`
- **THEN** that element receives the answer value

#### Scenario: 識別碼查無對應元素時跳過
- **WHEN** 答案的識別碼在三種解析方式下皆找不到對應元素
- **THEN** 該筆答案被跳過並記入 `skipped`，且其餘欄位仍持續填入，不中斷整批作業
- **WHEN** an answer's identifier resolves to no element under any of the three lookups
- **THEN** that answer is skipped and recorded in `skipped`, while the remaining fields continue to be filled without aborting the batch

#### Scenario: 填入 select 欄位
- **WHEN** 答案對應的欄位類型為 `select-one` 或 `select-multiple`
- **THEN** 該欄位的 `value` 被設為答案值，且答案值已通過選項白名單驗證
- **WHEN** an answer targets a `select-one` or `select-multiple` field
- **THEN** the field's `value` is set to the answer, which has already passed option-whitelist validation

#### Scenario: 勾選 radio 與 checkbox
- **WHEN** 答案值為字串且欄位類型為 `radio` 或 `checkbox`
- **THEN** 僅當答案值等於該欄位的 `value` 時將其設為勾選
- **WHEN** the answer is a string and the target field is a `radio` or `checkbox`
- **THEN** the field is checked only if the answer equals that field's `value`

#### Scenario: 以陣列處理多選欄位
- **WHEN** 答案值為字串陣列且欄位類型為 `checkbox` 或 `select-multiple`
- **THEN** 逐一比對後，勾選或選取所有命中項目，未命中者保持原狀
- **WHEN** the answer is an array of strings and the target is a `checkbox` or `select-multiple` field
- **THEN** each entry is compared, matching items are checked or selected, and non-matching items keep their prior state

#### Scenario: 尊重 maxlength 限制
- **WHEN** 答案值長度超過目標欄位的 `maxlength`
- **THEN** 該欄位不被填入，該筆答案記入 `skipped` 並於摘要中標示原因為超出長度限制
- **WHEN** an answer exceeds the target field's `maxlength`
- **THEN** the field is left unfilled, the answer is recorded in `skipped`, and the summary states the length-limit reason

### Requirement: 回填後派發 input 與 change 事件
After writing a value, the content script SHALL dispatch an `input` event and a `change` event on that element with `bubbles: true`, so that state-driven front-end frameworks observe the change.

#### Scenario: 派發事件使框架狀態同步
- **WHEN** 一個由 React 控制的 input 元素被填入值
- **THEN** content script 派發 `input` 與 `change` 事件後，該元素在框架中的狀態值與 DOM 值一致
- **WHEN** a React-controlled input element receives a filled value
- **THEN** after the content script dispatches the `input` and `change` events, the framework state value matches the DOM value

#### Scenario: 未派發事件的後果被禁止
- **WHEN** 實作僅設定 `el.value` 而未派發事件
- **THEN** 此行為違反本需求，視為未完成實作
- **WHEN** an implementation only sets `el.value` without dispatching events
- **THEN** this behavior violates this requirement and is treated as an incomplete implementation

### Requirement: 選擇型欄位以選項白名單驗證
The content script MUST NOT write a value into a `select`, `radio`, or `checkbox` field unless that value exists in the field's `options` set. Values that fail this check SHALL be skipped and recorded with a reason.

#### Scenario: 拒絕不在選項內的值
- **WHEN** 答案為 `"Premium"` 但目標 `select` 的 options 僅含 `"basic"` 與 `"pro"`
- **THEN** 該欄位不被填入，該筆答案記入 `skipped` 且原因為選項不符
- **WHEN** an answer is `"Premium"` but the target `select` only offers `"basic"` and `"pro"`
- **THEN** the field is left unfilled, the answer is recorded in `skipped` with an option-mismatch reason

#### Scenario: 支援以 option 文字輸入
- **WHEN** 答案值等於某個 `<option>` 的顯示文字而非其 `value`
- **THEN** content script 將該文字轉換為對應的 `value` 後填入
- **WHEN** an answer matches an `<option>`'s display text rather than its `value`
- **THEN** the content script converts that text to the corresponding `value` before filling

### Requirement: 敏感欄位留空並記錄
The content script MUST NOT write any LLM-generated value into fields identified as sensitive (identity, financial, or credential categories). It SHALL record each such field in `skipped` with a `sensitive_field` reason so the popup can list it for manual completion.

#### Scenario: 身分欄位不被填入
- **WHEN** 掃描結果中包含名為「身分證字號」的欄位，且後端回傳了該欄位的答案
- **THEN** content script 忽略該答案、不填入任何值，並以 `sensitive_field` 原因記入 `skipped`
- **WHEN** the scan includes a field named "身分證字號" (national ID) and the backend returns an answer for it
- **THEN** the content script ignores that answer, writes no value, and records it in `skipped` with a `sensitive_field` reason

#### Scenario: 財務欄位不被填入
- **WHEN** 掃描結果中包含信用卡號、CVV 或銀行帳號類欄位
- **THEN** 這些欄位一律不填入，並逐項以 `sensitive_field` 原因記入 `skipped`
- **WHEN** the scan includes credit card number, CVV, or bank account fields
- **THEN** none of them are filled, and each is recorded in `skipped` with a `sensitive_field` reason

#### Scenario: 敏感欄位清單可被使用者檢視
- **WHEN** 填寫作業完成
- **THEN** 所有 `sensitive_field` 項目均以欄位名稱列於摘要中，提示使用者手動填寫
- **WHEN** a fill operation completes
- **THEN** every `sensitive_field` entry is listed by field name in the summary, prompting the user to fill it manually
