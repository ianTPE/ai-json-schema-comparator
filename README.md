# 七大 AI 服務結構化 JSON 輸出對比

*最後更新：2025年9月10日*

## 概述

本文檔詳細比較了七個主要 AI 服務提供商在結構化 JSON 輸出方面的實現方式、參數配置、Schema 定義和使用複雜度。

## 適用場景比較

### 嚴格 Schema 驗證需求
**推薦順序**：OpenAI = Grok > Gemini > Claude > Qwen = Kimi-K2 > DeepSeek

適合需要精確數據結構驗證的場景，如 API 回應、數據庫記錄等。

### 開發便利性
**推薦順序**：DeepSeek > Claude > OpenAI = Grok = Gemini > Qwen = Kimi-K2

適合快速原型開發和簡單的結構化輸出需求。

### 工具調用整合
**推薦順序**：Qwen = Kimi-K2 > Claude > OpenAI > Gemini > Grok > DeepSeek

適合需要與現有工具系統整合的複雜應用。

### 成本效益
**推薦順序**：DeepSeek > Qwen > Kimi-K2 > Claude > Gemini > OpenAI > Grok

適合大規模部署和成本敏感的應用場景。

## 對比表格

| 特性             | OpenAI                            | Grok (XAI)                        | Gemini                                   | Claude (Anthropic)                   | Qwen                       | Kimi-K2                              | DeepSeek                             |
| -------------- | --------------------------------- | --------------------------------- | ---------------------------------------- | ----------------------------------- | -------------------------- | ------------------------------------ | ----------------------------------- |
| **主要方法**       | `response_format` + `json_schema` | `response_format` + `json_schema` | `response_mime_type` + `response_schema` | 工具調用 + Prefill + Prompt 引導        | 工具調用 (Function Calling)    | 工具調用 (Function Calling)             | `response_format` + `json_object`   |
| **參數名稱**       | `response_format.json_schema`     | `response_format.json_schema`     | `config.response_schema`                 | `tools` + `input_schema` / Prefill  | `tools` + `tool_choice`    | `tools` + `tool_choice`              | `response_format.type`              |
| **Schema 定義**  | JSON Schema 格式                    | JSON Schema 格式                    | JSON Schema 或 Pydantic 模型                | Tool input_schema / 自由格式          | Function parameters schema | Function parameters schema           | Prompt-guided JSON                 |
| **強制性**        | 強制 JSON Schema 驗證                 | 強制 JSON Schema 驗證                 | 強制 Schema 驗證                             | 工具調用：強制 / Prefill：建議性          | 工具調用：強制                    | 工具調用：強制                            | 基本 JSON 格式驗證                      |
| **錯誤處理**       | API 層面驗證                          | API 層面驗證                          | API 層面驗證                                 | 工具：API 驗證 / 其他：依賴提示         | API 層面驗證                   | API 層面驗證                            | 基本格式驗證                           |
| **開發複雜度**      | 低                                 | 低                                 | 低                                        | 低到中等                              | 中等                         | 中等                                  | 低                                   |
| **Schema 嚴格性** | 高 (`strict: true`)                | 高 (`strict: true`)                | 高                                        | 中等到高                              | 高                          | 中等                                  | 低到中等                              |

## 詳細實現說明

### 1. OpenAI

**實現方式**：
- 支援兩種 API 方式：
  - 新的 Responses API：`client.responses.create`（支援 GPT-5、GPT-4o、GPT-4o-mini 等）
  - 傳統 Chat Completions API：`client.chat.completions.create`（所有模型）
- 使用 `response_format` 參數結合嚴格的 JSON Schema
- 支援 `strict: true` 模式進行嚴格驗證

**配置範例**：
```python
# GPT-5 新的 Responses API
response = client.responses.create(
    model="gpt-5",
    input="您的輸入內容",
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "response_schema",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "age": {"type": "integer"}
                },
                "required": ["name", "age"],
                "additionalProperties": False
            }
        }
    }
)
```

### 2. Qwen

**實現方式**：
- 主要透過 Function Calling 實現結構化輸出
- 在工具定義中指定 JSON Schema
- **Qwen3-Max-Preview** 新增強化的指令遵循和結構化輸出能力

**最新能力提升（Qwen3-Max-Preview）**：
- 萬億參數規模，顯著提升複雜指令遵循能力
- 針對 RAG 和工具調用進行專門優化
- 更可靠的 Schema 驗證和減少幻覺
- 支援 256K tokens 上下文窗口

**配置範例**：
```python
tools = [{
    "type": "function",
    "function": {
        "name": "extract_info",
        "description": "提取資訊",
        "parameters": {
            "type": "object",
            "properties": {
                "name": {"type": "string"},
                "age": {"type": "integer"}
            },
            "required": ["name", "age"]
        }
    }
}]

response = client.chat.completions.create(
    model="qwen3-max-preview",  # 最新模型
    messages=messages,
    tools=tools,
    tool_choice="auto"
)
```

### 3. Grok (XAI)

**實現方式**：
- 類似 OpenAI，使用 `response_format` + `json_schema`
- 支援嚴格模式驗證

**配置範例**：
```python
response = client.chat.completions.create(
    model="grok-beta",
    messages=messages,
    response_format={
        "type": "json_schema",
        "json_schema": {
            "strict": True,
            "schema": schema_definition
        }
    }
)
```

### 4. Gemini

**實現方式**：
- 使用 `response_mime_type` 和 `response_schema`
- 支援 JSON Schema 或 Pydantic 模型

**配置範例**：
```python
generation_config = {
    "response_mime_type": "application/json",
    "response_schema": {
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "age": {"type": "integer"}
        },
        "required": ["name", "age"]
    }
}

response = model.generate_content(
    prompt,
    generation_config=generation_config
)
```

### 5. Kimi-K2

**實現方式**：
- 主要通過工具調用（Function Calling）實現
- 具備專門的工具調用解析器
- 完全兼容 OpenAI API 格式

**配置範例**：
```python
tools = [{
    "type": "function",
    "function": {
        "name": "extract_data",
        "description": "提取結構化數據",
        "parameters": {
            "type": "object",
            "properties": {
                "question": {"type": "string"},
                "answer": {"type": "string"}
            },
            "required": ["question", "answer"]
        }
    }
}]

response = client.chat.completions.create(
    model="kimi-k2",
    messages=messages,
    tools=tools,
    tool_choice="auto"
)
```

**特色功能**：
- 支援 `--tool-call-parser kimi_k2` 參數
- 提供手動解析工具調用的方法
- 支援流式和非流式模式

### 6. DeepSeek

**實現方式**：
- 使用 `response_format` 強制 JSON 輸出
- 基於 prompt 引導生成結構化內容
- 需要在 prompt 中提供 JSON 範例

**配置範例**：
```python
system_prompt = """
用戶會提供一些考試文本，請解析"問題"和"答案"並以 JSON 格式輸出。

範例輸入：
世界上最高的山是哪座？珠穆朗瑪峰。

範例 JSON 輸出：
{
    "question": "世界上最高的山是哪座？",
    "answer": "珠穆朗瑪峰"
}
"""

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_input}
    ],
    response_format={
        "type": "json_object"
    }
)
```

**使用要求**：
- prompt 中必須包含 "json" 關鍵字
- 需要提供 JSON 範例格式
- 適當設置 `max_tokens` 避免截斷

### 7. Claude (Anthropic)

**實現方式**：
- 提供多種結構化輸出方法：工具調用、預填充、提示引導
- 最靈活的實現方式，開發者可根據需求選擇合適方法
- 支援複雜的工具定義和自由格式的 JSON 輸出

**方法一：工具調用 (Tool Use)**
```python
tools = [
    {
        "name": "extract_info",
        "description": "Extract structured information from text",
        "input_schema": {
            "type": "object",
            "properties": {
                "name": {"type": "string", "description": "Person's name"},
                "age": {"type": "integer", "description": "Person's age"},
                "skills": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "List of skills"
                }
            },
            "required": ["name", "age"],
            "additionalProperties": True  # 允許額外欄位
        }
    }
]

response = client.messages.create(
    model="claude-3-sonnet-20240229",
    max_tokens=1024,
    tools=tools,
    tool_choice={"type": "tool", "name": "extract_info"},
    messages=[{"role": "user", "content": "Extract info from: John is 25 years old and knows Python."}]
)

# 提取工具使用結果
for content in response.content:
    if content.type == "tool_use":
        structured_data = content.input
        break
```

**方法二：預填充 (Prefill)**
```python
response = client.messages.create(
    model="claude-3-sonnet-20240229",
    max_tokens=1024,
    messages=[
        {
            "role": "user", 
            "content": "Extract name and age from: John is 25 years old. Output as JSON."
        },
        {
            "role": "assistant",
            "content": "{"  # 預填充強制 JSON 輸出
        }
    ]
)
```

**方法三：XML 標籤引導**
```python
response = client.messages.create(
    model="claude-3-sonnet-20240229",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": """Extract person info and put JSON in <person_data> tags.
        Text: John is 25 years old and works as an engineer."""
    }]
)

# 使用正則表達式提取 XML 標籤內的 JSON
import re
json_match = re.search(r'<person_data>(.*?)</person_data>', response.content[0].text, re.DOTALL)
if json_match:
    json_data = json.loads(json_match.group(1))
```

**特色功能**：
- **多方法支援**：根據使用場景選擇最適合的方法
- **靈活的 Schema**：支援 `additionalProperties: True` 允許未知欄位
- **自然語言引導**：通過詳細的描述引導模型輸出
- **組合使用**：可以同時使用多種方法提高成功率

**使用建議**：
- **嚴格結構化需求**：使用工具調用方法
- **簡單快速實現**：使用預填充方法
- **複雜多輸出場景**：使用 XML 標籤方法
- **探索性數據提取**：使用 `additionalProperties: True`

適合大規模部署和成本敏感的應用場景。

## 最佳實踐建議

### 1. Schema 設計
- 明確定義所有必需欄位
- 使用描述性的欄位名稱
- 適當使用 `additionalProperties: false` 限制額外欄位

### 2. 錯誤處理
- 實施回退機制處理 Schema 驗證失敗
- 記錄和監控輸出格式錯誤
- 設計容錯的解析邏輯

### 3. 效能優化
- 選擇適合的 Schema 複雜度
- 合理設置 token 限制
- 考慮快取常用的 Schema 定義

### 4. 測試策略
- 建立全面的測試案例集
- 測試邊界條件和異常輸入
- 驗證不同模型版本的一致性

## 總結

不同 AI 服務在結構化 JSON 輸出方面各有特色：

- **OpenAI 和 Grok** 提供最嚴格的 Schema 驗證
- **Qwen 和 Kimi-K2** 通過工具調用實現靈活的結構化輸出，Qwen3-Max-Preview 具備萬億參數規模和增強的 Schema 驗證能力
- **Gemini** 提供多種 Schema 定義方式
- **Claude** 提供多種實現方法，靈活性最高，支援工具調用、預填充和提示引導
- **DeepSeek** 以簡單易用為特色，適合快速開發

選擇時應根據具體需求、開發複雜度、成本預算和精確度要求來決定最適合的方案。

---

*最後更新日期：2025年9月*