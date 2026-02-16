# APItemplateV2
ForkSilly应用的API模板仓库。

# API 模板制作说明书

## 概述

模板引擎通过一个 JSON 配置文件来描述任意 LLM API 的请求/响应结构。引擎读取配置后自动完成：URL 拼接、请求头构建、请求体填充、采样参数映射、图片注入、流式/非流式响应解析等全部工作。

模板文件是一个 `.json` 文件，顶层结构如下：

```json
{
  "version": 2,
  "name": "模板显示名称",
  "connection": { ... },
  "request": { ... },
  "response": { ... },
  "model": { ... },
  "features": { ... },
  "uiOverridableSamplers": [ ... ],
  "media": { ... }
}
```

---

## 1. 基础字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `version` | `number` | ✅ | 必须为 `2` |
| `name` | `string` | ✅ | 模板名称，显示在 UI 中 |

---

## 2. `connection` — 连接配置

定义 API 的端点地址和认证方式。

```json
"connection": {
  "endpoint": "/v1/chat/completions",
  "streamEndpoint": "/v1/chat/completions",
  "modelEndpoint": "/v1/models",
  "headers": {
    "Content-Type": "application/json"
  },
  "auth": {
    "header": "Authorization",
    "prefix": "Bearer ",
    "extraHeaders": {}
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `endpoint` | `string` | ✅ | 非流式请求的端点路径，支持 `{{model}}` 宏 |
| `streamEndpoint` | `string` | ❌ | 流式请求的端点路径。不填则复用 `endpoint` |
| `modelEndpoint` | `string` | ❌ | 获取模型列表的端点路径，默认 `/models` |
| `headers` | `object` | ❌ | 额外请求头，会合并到最终请求中 |
| `auth` | `object` | ❌ | 认证配置。不填则默认使用 `Authorization: Bearer <key>` |
| `auth.header` | `string` | — | 认证头名称，如 `"Authorization"` 或 `"x-goog-api-key"` |
| `auth.prefix` | `string` | — | 认证值前缀。默认 `"Bearer "`。设为 `""` 表示直接使用 key 无前缀 |
| `auth.extraHeaders` | `object` | — | 额外的认证相关头 |

**端点路径中的宏替换：** `{{model}}` 会被替换为用户选择的模型名称。这对 Gemini 这类将模型名嵌入 URL 的 API 很有用：

```json
"endpoint": "/v1beta/models/{{model}}:generateContent"
```

最终 URL = 用户填写的 `apiUrl`（去掉末尾斜杠）+ 端点路径。例如用户填 `https://generativelanguage.googleapis.com`，最终为：
`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`

---

## 3. `request` — 请求配置

定义请求体的结构、采样参数映射、prompt 格式。

### 3.1 `bodyTemplate` — 请求体模板（必填）

这是请求体的"骨架"。引擎会深拷贝它，然后往里面填充 prompt、采样参数等。你可以在里面放任何 API 需要的固定字段。

```json
"bodyTemplate": {
  "contents": [],
  "generationConfig": {
    "temperature": 0.7,
    "maxOutputTokens": 2048
  },
  "safetySettings": [
    { "category": "HARM_CATEGORY_HARASSMENT", "threshold": "BLOCK_NONE" }
  ]
}
```

规则：
- `bodyTemplate` 中的固定值会原样保留在最终请求中
- 如果某个采样参数在 `samplerMappings` 中声明了，且用户在 UI 中启用了该参数，则 UI 值会覆盖 `bodyTemplate` 中的对应值
- 如果用户未启用，则保留 `bodyTemplate` 中的默认值
- 不在 `samplerMappings` 中的字段永远保持原样（如上面的 `safetySettings`）

### 3.2 `samplerMappings` — 采样参数映射（可选）

将应用内的标准采样参数映射到请求体中的具体路径。

```json
"samplerMappings": [
  { "samplerID": "temperature", "path": "$.generationConfig.temperature" },
  { "samplerID": "maxTokens", "path": "$.generationConfig.maxOutputTokens", "transform": "integer" },
  { "samplerID": "topP", "path": "$.generationConfig.topP" },
  { "samplerID": "topK", "path": "$.generationConfig.topK", "transform": "integer" }
]
```

| 字段 | 说明 |
|------|------|
| `samplerID` | 标准参数 ID，可选值见下表 |
| `path` | 写入请求体的 JSONPath 路径 |
| `transform` | 可选的值转换：`"integer"`（取整）、`"string"`（转字符串）、`"boolean"`（转布尔） |

可用的 `samplerID`：

| samplerID | 说明 |
|-----------|------|
| `temperature` | 温度 |
| `maxTokens` | 最大输出 token 数 |
| `topP` | Top-P 采样 |
| `topK` | Top-K 采样 |
| `frequencyPenalty` | 频率惩罚 |
| `presencePenalty` | 存在惩罚 |
| `reasoningEffort` | 推理强度 |

### 3.3 `stop` — 停止序列配置（可选）

```json
"stop": {
  "path": "$.generationConfig.stopSequences",
  "limit": 5
}
```

应用中没有开放停止序列的配置，如果有有需求，请将固定值配置到body中，不要使用这里的配置。

| 字段 | 说明 |
|------|------|
| `path` | 停止序列写入请求体的 JSONPath |
| `limit` | 最大停止序列数量 |

### 3.4 `promptPath` — Prompt 写入路径（必填）

指定格式化后的消息数组写入请求体的哪个位置。

```json
"promptPath": "$.contents"
```

- OpenAI 格式通常是 `"$.messages"`
- Gemini 格式通常是 `"$.contents"`

### 3.5 `promptFormat` — Prompt 格式（必填）

定义消息的格式化方式。支持两种类型：

#### `chat` 类型（对话格式）

```json
"promptFormat": {
  "type": "chat",
  "roles": {
    "user": "user",
    "assistant": "model",
    "system": "user"
  },
  "contentKey": "parts"
}
```

| 字段 | 说明 |
|------|------|
| `type` | 固定为 `"chat"` |
| `roles.user` | user 角色在 API 中的名称。OpenAI 为 `"user"`，Gemini 为 `"user"` |
| `roles.assistant` | assistant 角色在 API 中的名称。OpenAI 为 `"assistant"`，Gemini 为 `"model"` |
| `roles.system` | system 角色在 API 中的名称。Gemini 不支持 system role，映射为 `"user"` |
| `contentKey` | 消息内容字段名。OpenAI 为 `"content"`，Gemini 为 `"parts"` |

引擎会将内部消息格式转换为 API 要求的格式。例如 Gemini 模板会生成：

```json
[
  { "role": "user", "parts": [{ "text": "你好" }] },
  { "role": "model", "parts": [{ "text": "你好！" }] }
]
```

#### `text` 类型（文本补全格式）

```json
"promptFormat": {
  "type": "text"
}
```

所有消息会被拼接为 `"role: content\nrole: content\n..."` 的纯文本字符串。

---

## 4. `response` — 响应配置

定义如何从 API 响应中提取内容。

```json
"response": {
  "transport": { "type": "sse" },
  "contentPath": "$.candidates[0].content.parts[0].text",
  "streamContentPath": "$.candidates[0].content.parts[0].text",
  "reasoningPath": "$.candidates[0].content.parts[0].thought",
  "streamReasoningPath": "$.candidates[0].content.parts[0].thought",
  "error": {
    "detectPath": "$.error",
    "messagePath": "$.error.message"
  }
}
```

### 4.1 `transport` — 传输方式（必填）

| type | 说明 |
|------|------|
| `"sse"` | Server-Sent Events 流式传输 |
| `"fetch"` | 普通 HTTP 请求，等待完整响应 |
| `"polling"` | 轮询模式，适用于异步任务型 API（未测试） |

#### SSE 传输配置

```json
"transport": {
  "type": "sse",
  "format": "standard",
  "doneSignal": "[DONE]"
}
```

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `format` | SSE 数据格式：`"standard"` / `"ndjson"` / `"auto"` | `"standard"` |
| `doneSignal` | 流结束信号 | `"[DONE]"` |

#### Polling 传输配置

```json
"transport": {
  "type": "polling",
  "submitPath": "/v1/tasks",
  "statusPath": "/v1/tasks/{{taskId}}",
  "taskIdPath": "$.task_id",
  "donePath": "$.status.completed",
  "interval": 2000
}
```

| 字段 | 说明 |
|------|------|
| `submitPath` | 提交任务的端点路径 |
| `statusPath` | 查询任务状态的端点路径，支持 `{{taskId}}` 宏 |
| `taskIdPath` | 从提交响应中提取任务 ID 的 JSONPath |
| `donePath` | 判断任务完成的 JSONPath（值为 truthy 时视为完成） |
| `interval` | 轮询间隔（毫秒），默认 2000 |

### 4.2 内容提取路径

| 字段 | 必填 | 说明 |
|------|------|------|
| `contentPath` | ✅ | 从非流式响应中提取内容的 JSONPath |
| `streamContentPath` | ❌ | 从流式响应中提取内容的 JSONPath。不填则复用 `contentPath` |
| `reasoningPath` | ❌ | 从非流式响应中提取推理/思考内容的 JSONPath |
| `streamReasoningPath` | ❌ | 从流式响应中提取推理/思考内容的 JSONPath |

**常见 API 的路径对照：**

| API | contentPath | streamContentPath |
|-----|-------------|-------------------|
| OpenAI | `$.choices[0].message.content` | `$.choices[0].delta.content` |
| Gemini | `$.candidates[0].content.parts[0].text` | `$.candidates[0].content.parts[0].text` |
| Claude | `$.content[0].text` | `$.delta.text` |

### 4.3 `error` — 错误检测（可选）

```json
"error": {
  "detectPath": "$.error",
  "messagePath": "$.error.message"
}
```

| 字段 | 说明 |
|------|------|
| `detectPath` | 错误检测路径。该路径的值存在且为 truthy 时视为错误 |
| `messagePath` | 错误消息的 JSONPath |

不配置时，引擎会默认检查响应中的 `error` 字段。

---

## 5. `model` — 模型列表解析配置（可选）

定义如何解析模型列表 API 的响应。

```json
"model": {
  "useModelContextLength": true,
  "listPath": "$.models",
  "namePath": "$.name",
  "contextLengthPath": "$.inputTokenLimit"
}
```

| 字段 | 说明 |
|------|------|
| `useModelContextLength` | 是否使用模型返回的上下文长度 |
| `listPath` | 模型列表数组的 JSONPath，如 OpenAI 的 `"$.data"`，Gemini 的 `"$.models"` |
| `namePath` | 模型名称的 JSONPath（相对于列表中的每个元素），如 `"$.id"` 或 `"$.name"` |
| `contextLengthPath` | 上下文长度的 JSONPath（相对于列表中的每个元素） |

不配置时，引擎会尝试自动识别 OpenAI 格式（`$.data[].id`）和其他常见格式。

---

## 6. `features` — 功能开关（目前不会影响UI）

控制 UI 上哪些功能可见。

```json
"features": {
  "useKey": true,
  "useModel": true,
  "useStreaming": true
}
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `useKey` | `true` | 是否需要 API Key |
| `useModel` | `true` | 是否需要模型选择 |
| `useStreaming` | `true` | 是否允许切换流式/非流式 |

---

## 7. `uiOverridableSamplers` — UI 可调参数（可选）

列出哪些采样参数在 UI 上显示为可配置。未列出的参数在 UI 上隐藏，使用 `bodyTemplate` 中的固定值。(目前实际不会在UI上隐藏，配置需要的参数即可)

```json
"uiOverridableSamplers": ["temperature", "maxTokens", "topP", "topK"]
```

可选值与 `samplerID` 相同：`temperature`、`maxTokens`、`topP`、`topK`、`frequencyPenalty`、`presencePenalty`、`reasoningEffort`。

---

## 8. `media` — 媒体处理配置（可选）

定义图片发送的格式。

```json
"media": {
  "supportsImages": true,
  "imageContentTemplate": {
    "inlineData": {
      "mimeType": "{{mime_type}}",
      "data": "{{image_base64}}"
    }
  },
  "textContentTemplate": {
    "text": "{{text}}"
  }
}
```

| 字段 | 说明 |
|------|------|
| `supportsImages` | 是否支持发送图片，默认 `false` |
| `imageContentTemplate` | 图片在消息中的格式模板，支持 `{{image_base64}}` 和 `{{mime_type}}` 宏 |
| `textContentTemplate` | 文本在消息中的格式模板（当消息同时包含文本和图片时使用），支持 `{{text}}` 宏 |

**不同 API 的图片格式对照：**

OpenAI：
```json
"imageContentTemplate": {
  "type": "image_url",
  "image_url": { "url": "data:{{mime_type}};base64,{{image_base64}}" }
},
"textContentTemplate": {
  "type": "text",
  "text": "{{text}}"
}
```

Gemini：
```json
"imageContentTemplate": {
  "inlineData": { "mimeType": "{{mime_type}}", "data": "{{image_base64}}" }
},
"textContentTemplate": {
  "text": "{{text}}"
}
```

Claude：
```json
"imageContentTemplate": {
  "type": "image",
  "source": { "type": "base64", "media_type": "{{mime_type}}", "data": "{{image_base64}}" }
},
"textContentTemplate": {
  "type": "text",
  "text": "{{text}}"
}
```

当 `textContentTemplate` 存在时，纯文本消息的 content 也会被包装为数组格式。例如 Gemini 模板下，`"你好"` 会变成 `[{ "text": "你好" }]`。

不配置 `media` 或 `supportsImages` 为 `false` 时，图片功能关闭。

---

## JSONPath 语法说明

模板中所有 `path` 字段使用简化的 JSONPath 语法：

| 语法 | 含义 | 示例 |
|------|------|------|
| `$` | 根对象 | `$` |
| `.key` | 对象属性 | `$.temperature` |
| `[0]` | 数组索引 | `$.choices[0]` |
| `["key"]` | 带引号的属性 | `$.["special-key"]` |

组合示例：`$.candidates[0].content.parts[0].text` 表示 `obj.candidates[0].content.parts[0].text`。

写入时，如果中间路径不存在，引擎会自动创建对象或数组。

---

## 宏替换说明

模板中的字符串值支持 `{{macro}}` 宏替换：

| 宏 | 可用位置 | 说明 |
|----|----------|------|
| `{{model}}` | `endpoint`、`streamEndpoint`、`bodyTemplate` 中的字符串 | 替换为用户选择的模型名称 |
| `{{text}}` | `textContentTemplate` | 替换为消息文本内容 |
| `{{image_base64}}` | `imageContentTemplate` | 替换为图片的 base64 数据 |
| `{{mime_type}}` | `imageContentTemplate` | 替换为图片的 MIME 类型 |
| `{{taskId}}` | `polling.statusPath` | 替换为轮询任务 ID |

---

## 完整示例

### 示例 1：OpenAI 兼容格式

```json
{
  "version": 2,
  "name": "OpenAI Compatible",
  "connection": {
    "endpoint": "/v1/chat/completions",
    "modelEndpoint": "/v1/models",
    "auth": {
      "header": "Authorization",
      "prefix": "Bearer "
    }
  },
  "request": {
    "bodyTemplate": {
      "model": "{{model}}",
      "messages": [],
      "stream": false
    },
    "samplerMappings": [
      { "samplerID": "temperature", "path": "$.temperature" },
      { "samplerID": "maxTokens", "path": "$.max_tokens", "transform": "integer" },
      { "samplerID": "topP", "path": "$.top_p" },
      { "samplerID": "frequencyPenalty", "path": "$.frequency_penalty" },
      { "samplerID": "presencePenalty", "path": "$.presence_penalty" }
    ],
    "promptPath": "$.messages",
    "promptFormat": {
      "type": "chat",
      "roles": { "user": "user", "assistant": "assistant", "system": "system" },
      "contentKey": "content"
    }
  },
  "response": {
    "transport": { "type": "sse" },
    "contentPath": "$.choices[0].message.content",
    "streamContentPath": "$.choices[0].delta.content",
    "error": {
      "detectPath": "$.error",
      "messagePath": "$.error.message"
    }
  },
  "model": {
    "listPath": "$.data",
    "namePath": "$.id"
  },
  "features": {
    "useKey": true,
    "useModel": true,
    "useStreaming": true
  },
  "uiOverridableSamplers": ["temperature", "maxTokens", "topP", "frequencyPenalty", "presencePenalty"],
  "media": {
    "supportsImages": true,
    "imageContentTemplate": {
      "type": "image_url",
      "image_url": { "url": "data:{{mime_type}};base64,{{image_base64}}" }
    },
    "textContentTemplate": {
      "type": "text",
      "text": "{{text}}"
    }
  }
}
```

### 示例 2：Google Gemini

```json
{
  "version": 2,
  "name": "Google Gemini (Native API)",
  "connection": {
    "endpoint": "/v1beta/models/{{model}}:generateContent",
    "streamEndpoint": "/v1beta/models/{{model}}:streamGenerateContent?alt=sse",
    "modelEndpoint": "/v1beta/models",
    "headers": { "Content-Type": "application/json" },
    "auth": { "header": "x-goog-api-key", "prefix": "" }
  },
  "request": {
    "bodyTemplate": {
      "contents": [],
      "generationConfig": {
        "temperature": 0.7,
        "maxOutputTokens": 2048,
        "topP": 0.95,
        "topK": 40
      },
      "safetySettings": [
        { "category": "HARM_CATEGORY_HARASSMENT", "threshold": "BLOCK_NONE" },
        { "category": "HARM_CATEGORY_HATE_SPEECH", "threshold": "BLOCK_NONE" },
        { "category": "HARM_CATEGORY_SEXUALLY_EXPLICIT", "threshold": "BLOCK_NONE" },
        { "category": "HARM_CATEGORY_DANGEROUS_CONTENT", "threshold": "BLOCK_NONE" },
        { "category": "HARM_CATEGORY_CIVIC_INTEGRITY", "threshold": "BLOCK_NONE" }
      ]
    },
    "samplerMappings": [
      { "samplerID": "temperature", "path": "$.generationConfig.temperature" },
      { "samplerID": "maxTokens", "path": "$.generationConfig.maxOutputTokens", "transform": "integer" },
      { "samplerID": "topP", "path": "$.generationConfig.topP" },
      { "samplerID": "topK", "path": "$.generationConfig.topK", "transform": "integer" }
    ],
    "stop": { "path": "$.generationConfig.stopSequences", "limit": 5 },
    "promptPath": "$.contents",
    "promptFormat": {
      "type": "chat",
      "roles": { "user": "user", "assistant": "model", "system": "user" },
      "contentKey": "parts"
    }
  },
  "response": {
    "transport": { "type": "sse" },
    "contentPath": "$.candidates[0].content.parts[0].text",
    "streamContentPath": "$.candidates[0].content.parts[0].text",
    "reasoningPath": "$.candidates[0].content.parts[0].thought",
    "streamReasoningPath": "$.candidates[0].content.parts[0].thought",
    "error": { "detectPath": "$.error", "messagePath": "$.error.message" }
  },
  "model": {
    "useModelContextLength": true,
    "listPath": "$.models",
    "namePath": "$.name",
    "contextLengthPath": "$.inputTokenLimit"
  },
  "features": { "useKey": true, "useModel": true, "useStreaming": true },
  "uiOverridableSamplers": ["temperature", "maxTokens", "topP", "topK"],
  "media": {
    "supportsImages": true,
    "imageContentTemplate": {
      "inlineData": { "mimeType": "{{mime_type}}", "data": "{{image_base64}}" }
    },
    "textContentTemplate": { "text": "{{text}}" }
  }
}
```

---

## 制作新模板的步骤

1. 阅读目标 API 的官方文档，了解请求/响应格式
2. 复制一个最接近的示例模板作为起点
3. 修改 `connection` 中的端点路径和认证方式
4. 修改 `bodyTemplate`，放入 API 要求的固定字段
5. 配置 `samplerMappings`，将标准参数映射到 `bodyTemplate` 中的正确路径
6. 配置 `promptPath` 和 `promptFormat`，确保消息格式正确
7. 配置 `response` 中的内容提取路径（可以先用非流式测试，再配流式路径）
8. 如需图片支持，配置 `media` 部分
9. 保存为 `.json` 文件，在应用中导入测试

**调试技巧：** 如果响应解析不到内容，先用非流式模式测试，观察 API 返回的完整 JSON 结构，然后调整 `contentPath`。流式模式下每个 SSE 事件的 JSON 结构可能与非流式不同（如 OpenAI 的 `delta` vs `message`），需要分别配置 `streamContentPath`。
