這是一個基於 Chrome Extension V3 架構的初版外掛程式碼。這版程式碼分為四個主要部分：定義檔 (Manifest)、擷取與填寫網頁內容的腳本 (Content Script)、處理外部 LiteLLM API 請求的背景腳本 (Background Script)，以及觸發功能的使用者介面 (Popup)。

### **1\. manifest.json (擴充功能設定檔)**

這是 Chrome 外掛的核心設定檔，宣告所需的權限與載入的腳本。

JSON  
{  
  "manifest\_version": 3,  
  "name": "Form AI Auto-Filler",  
  "version": "1.0",  
  "description": "Analyze page elements and auto-fill forms using LiteLLM. / 使用 LiteLLM 分析網頁元素並自動填寫表單。",  
  "permissions": \[  
    "activeTab",  
    "scripting",  
    "storage"  
  \],  
  "background": {  
    "service\_worker": "background.js"  
  },  
  "action": {  
    "default\_popup": "popup.html"  
  },  
  "content\_scripts": \[  
    {  
      "matches": \["\<all\_urls\>"\],  
      "js": \["content.js"\]  
    }  
  \]  
}

### **2\. popup.html (使用者介面)**

點擊擴充功能圖示時彈出的簡單介面，包含一個觸發按鈕。

HTML  
\<\!DOCTYPE **html**\>  
\<html\>  
\<head\>  
  \<meta charset\="UTF-8"\>  
  \<style\>  
    body { width: 250px; padding: 10px; font-family: Arial, sans-serif; }  
    button { width: 100%; padding: 10px; margin-top: 10px; cursor: pointer; }  
  \</style\>  
\</head\>  
\<body\>  
  \<h3\>AI 表單助手 / AI Form Helper\</h3\>  
  \<button id\="fillBtn"\>分析並填寫 / Analyze & Fill\</button\>  
  \<div id\="status" style\="margin-top: 10px; font-size: 12px; color: gray;"\>\</div\>  
  \<script src\="popup.js"\>\</script\>  
\</body\>  
\</html\>

### **3\. popup.js (介面互動邏輯)**

負責將點擊事件傳遞給目前活躍的網頁 (Content Script)。

JavaScript  
document.getElementById('fillBtn').addEventListener('click', async () \=\> {  
  const statusDiv \= document.getElementById('status');  
  statusDiv.innerText \= "Analyzing page... / 正在分析網頁...";

  // 取得當前活躍的標籤頁 / Get current active tab  
  let \[tab\] \= await chrome.tabs.query({ active: true, currentWindow: true });

  // 傳送訊息給 content.js 開始處理 / Send message to content.js to start processing  
  chrome.tabs.sendMessage(tab.id, { action: "START\_FILL" }, (response) \=\> {  
    if (chrome.runtime.lastError) {  
      statusDiv.innerText \= "Error: Please refresh the page. / 錯誤：請重整網頁。";  
      return;  
    }  
    statusDiv.innerText \= response.status || "Process finished. / 處理完成。";  
  });  
});

### **4\. content.js (網頁解析與自動填寫邏輯)**

這支程式碼負責找出網頁上所有的輸入元素（文字框、單選、多選、下拉選單），將它們轉換為結構化資料，傳送給背景腳本，收到答案後進行填寫。

JavaScript  
// 監聽來自 popup 的訊息 / Listen for messages from popup  
chrome.runtime.onMessage.addListener((request, sender, sendResponse) \=\> {  
  if (request.action \=== "START\_FILL") {  
    processForm();  
    sendResponse({ status: "Processing... / 正在處理中..." });  
  }  
  return true;   
});

async function processForm() {  
  const formElements \= \[\];  
  // 尋找所有可能的輸入欄位 / Find all possible input fields  
  const inputs \= document.querySelectorAll('input:not(\[type="hidden"\]):not(\[type="submit"\]), textarea, select');

  inputs.forEach((el, index) \=\> {  
    // 確保每個元素都有唯一的 ID 或 Name 供識別 / Ensure element has an ID or Name for mapping  
    const identifier \= el.id || el.name || \`ai\_input\_\${index}\`;  
    if (\!el.id) el.setAttribute('data-ai-id', identifier); // 標記未命名元素 / Mark unnamed elements

    // 嘗試抓取附近的文字作為「問題」/ Try to extract nearby text as the "question"  
    let labelText \= "";  
    if (el.labels && el.labels.length \> 0) {  
      labelText \= el.labels\[0\].innerText;  
    } else {  
      // 若無標籤，抓取父層或前一個兄弟節點的文字 / If no label, get parent or previous sibling text  
      labelText \= el.parentElement.innerText || "";  
    }

    // 收集選項 (針對下拉選單或單/多選) / Collect options for selects/radios/checkboxes  
    let options \= \[\];  
    if (el.tagName.toLowerCase() \=== 'select') {  
      options \= Array.from(el.options).map(opt \=\> opt.value || opt.innerText);  
    } else if (el.type \=== 'radio' || el.type \=== 'checkbox') {  
      options.push(el.value);  
    }

    formElements.push({  
      id: el.id || el.getAttribute('data-ai-id'),  
      type: el.type || el.tagName.toLowerCase(),  
      questionText: labelText.substring(0, 100).trim(), // 限制長度 / Limit length  
      options: options.length \> 0 ? options : null  
    });  
  });

  // 傳送結構化資料給 Background Script 呼叫 API / Send data to Background to call API  
  chrome.runtime.sendMessage({ action: "CALL\_LITELLM", data: formElements }, (response) \=\> {  
    if (response && response.answers) {  
      fillForm(response.answers);  
    }  
  });  
}

function fillForm(answers) {  
  // answers 格式預期為: { "identifier": "回答內容" } / Expected format: { "id": "answer" }  
  for (const \[id, value\] of Object.entries(answers)) {  
    const el \= document.getElementById(id) || document.querySelector(\`\[name="\${id}"\]\`) || document.querySelector(\`\[data-ai-id="\${id}"\]\`);  
    if (\!el) continue;

    const type \= el.type || el.tagName.toLowerCase();

    // 根據不同類型進行填寫 / Fill based on input type  
    if (type \=== 'radio' || type \=== 'checkbox') {  
      // 處理單選與多選 / Handle radio and checkbox  
      if (Array.isArray(value) ? value.includes(el.value) : el.value \=== value) {  
        el.checked \= true;  
      }  
    } else if (type \=== 'select-one' || type \=== 'select-multiple') {  
      // 處理下拉選單 / Handle dropdowns  
      el.value \= value;  
    } else {  
      // 處理一般文字與 TextArea / Handle text and textarea  
      el.value \= value;  
    }

    // 觸發事件以確保前端框架 (React/Vue) 能抓到變更 / Dispatch events for modern frontend frameworks  
    el.dispatchEvent(new Event('input', { bubbles: true }));  
    el.dispatchEvent(new Event('change', { bubbles: true }));  
  }  
}

### **5\. background.js (呼叫 LiteLLM API)**

負責處理與 LiteLLM 伺服器的通訊，避免 Content Script 受到跨來源資源共用 (CORS) 的限制。請將 API\_URL 與 API\_KEY 換成您實際使用的 LiteLLM 節點。

JavaScript  
// 這裡設定您的 LiteLLM 伺服器網址與金鑰 / Set your LiteLLM API URL and Key here  
const LITELLM\_API\_URL \= "http://localhost:4000/v1/chat/completions";   
const LITELLM\_API\_KEY \= "sk-your-litellm-key";  
const MODEL\_NAME \= "gpt-3.5-turbo"; // 根據您的 LiteLLM 配置更改模型 / Change model based on your LiteLLM config

chrome.runtime.onMessage.addListener((request, sender, sendResponse) \=\> {  
  if (request.action \=== "CALL\_LITELLM") {  
    fetchAnswersFromLLM(request.data)  
      .then(answers \=\> sendResponse({ answers: answers }))  
      .catch(error \=\> {  
        console.error("LLM API Error:", error);  
        sendResponse({ answers: {} });  
      });  
    return true; // 保持異步訊息通道開啟 / Keep asynchronous message channel open  
  }  
});

async function fetchAnswersFromLLM(formElements) {  
  // 設計給 LLM 的提示詞，要求回傳 JSON 格式 / Prompt design asking LLM to return JSON  
  const systemPrompt \= \`  
    You are an AI assistant helping to fill out a web form.   
    I will provide a JSON array of form elements found on a page. Each element has an 'id', 'type', 'questionText', and 'options'.  
    Your task is to analyze the questions and provide the best logical answer for each.  
      
    Respond ONLY with a valid JSON object where the key is the element's 'id' and the value is the answer.  
    \- For 'text' or 'textarea', provide a generated string.  
    \- For 'radio', 'checkbox', or 'select', provide one of the available 'options' exactly as written.  
    \- If it's a multi-select or checkbox with multiple answers, provide an array of strings.  
      
    DO NOT wrap the JSON in markdown code blocks. Output purely the JSON object.  
      
    中文指示：  
    你是一個幫助填寫網頁表單的 AI。我會提供表單元素的 JSON。  
    請分析問題並給出合理的答案。只能回傳純 JSON 格式，Key 為元素 'id'，Value 為您的答案。  
    若為選擇題，答案必須完全符合選項內容。不要包含 markdown 標籤。  
  \`;

  const payload \= {  
    model: MODEL\_NAME,  
    messages: \[  
      { role: "system", content: systemPrompt },  
      { role: "user", content: JSON.stringify(formElements) }  
    \],  
    temperature: 0.1, // 降低溫度以確保格式穩定 / Low temperature for stable output  
    response\_format: { type: "json\_object" } // 強制 JSON 輸出 (若模型支援) / Force JSON if supported  
  };

  const response \= await fetch(LITELLM\_API\_URL, {  
    method: 'POST',  
    headers: {  
      'Content-Type': 'application/json',  
      'Authorization': \`Bearer \${LITELLM\_API\_KEY}\`  
    },  
    body: JSON.stringify(payload)  
  });

  const data \= await response.json();  
  try {  
    // 解析 LLM 回傳的字串為 JSON 物件 / Parse LLM response string to JSON object  
    const answers \= JSON.parse(data.choices\[0\].message.content);  
    return answers;  
  } catch (e) {  
    console.error("Failed to parse LLM response:", e);  
    return {};  
  }  
}

### **後續測試步驟**

> 1. 將上述 5 個檔案儲存在同一個資料夾中（例如命名為 Form-AI-Filler）。  
> 2. 開啟 Chrome 瀏覽器，前往 chrome://extensions/。  
> 3. 打開右上角的 **「開發人員模式 (Developer mode)」**。  
> 4. 點擊左上角的 **「載入未封裝項目 (Load unpacked)」**，並選擇該資料夾。  
> 5. 開啟您的目標測試站台，修改 background.js 中的 LITELLM\_API\_URL 與模型名稱以符合您本地/雲端的 LiteLLM 伺服器設定。  
> 6. 點擊瀏覽器右上角的外掛圖示並執行。