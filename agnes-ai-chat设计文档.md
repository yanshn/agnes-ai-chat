# Agnes AI Chat - 对话模式功能设计文档

## 概述

在现有生图、生视频两种模式基础上，新增 AI 对话文字模式，支持流式输出、多轮对话、深度思考、工具调用等能力。

对接 **Agnes-2.0-Flash** 对话模型：
- API 文档：https://agnes-ai.com/doc/agnes-20-flash
- Endpoint: `POST https://apihub.agnes-ai.com/v1/chat/completions`
- Auth: `Authorization: Bearer YOUR_API_KEY`
- Context Window: 256K
- Max Output: 65.5K

---

## 一、模式架构

### 1.1 三模式切换

```
🎨 图片  → 生图 API（/v1/image/generations）
🎬 视频  → 生视频 API（/v1/video/generations）
💬 对话  → 对话 API（/v1/chat/completions）[新增]
```

- Chat 模式按钮从"图片/视频"二选一扩展为"图片/视频/对话"三选一
- 每个对话独立记忆 chatMode (`conv.chatMode`)
- 选择「对话」模式后自动加载对话配置

### 1.2 对话模式下的 UI 变化

| 元素 | 图片/视频模式 | 对话模式 |
|------|:---:|:---:|
| 参考图 URL 输入框 | 显示 | **显示** |
| 参考图缩略图 | 显示 | **显示** |
| 提示词 placeholder | "描述你想要生成的图像……" | "输入消息……（可附带图片URL）" |
| 历史消息显示 | 图片/视频网格 | **对话气泡形式** |
| 操作按钮（删除/重生成/复制/请求体） | 显示 | **简化为仅复制** |

---

## 二、侧边栏：新增「对话配置」

```
┌─ 通用配置 ──────────────────────┐
│ API Key / 随机种子                │
├─ 生图配置 ──────────────────────┤
│ 模型 / 输出尺寸                   │
├─ 生视频配置 ────────────────────┤
│ 模型 / 分辨率 / 帧数 / 帧率        │
├─ 对话配置 ──────── [新增] ──────┤
│ 模型: [下拉] agnes-2.0-flash     │
│ 系统提示词: [textarea, 3行]       │
│ 温度 (temperature): [range 0~2]   │
│ 最大 Token: [number]              │
│ 深度思考: [checkbox toggle]        │
├─ 说明 ──────────────────────────┤
│ Agnes 平台介绍                   │
└─────────────────────────────────┘
```

### 2.1 对话配置项详情

| 设置项 | DOM id | localStorage key | 默认值 | 说明 |
|--------|--------|-----------------|--------|------|
| 模型 | `chatModel` | `agnes_chat_model` | `agnes-2.0-flash` | 目前仅一个 |
| 系统提示词 | `chatSystem` | `agnes_chat_system` | `""` | system role 消息 |
| 温度 | `chatTemp` | `agnes_chat_temp` | `0.7` | 0=确定，1=随机 |
| 最大 Token | `chatMaxTokens` | `agnes_chat_max_tokens` | `4096` | 1024~65535 |
| 深度思考 | `chatThinking` | `agnes_chat_thinking` | `false` | 启用 thinking 模式 |

### 2.2 DOM 元素设计

```html
<div class="sidebar-section">
  <h3 class="sidebar-group-title">对话配置</h3>
</div>

<div class="sidebar-section">
  <label class="sidebar-label">模型</label>
  <select id="chatModel">
    <option value="agnes-2.0-flash">Agnes-2.0-Flash</option>
  </select>
</div>

<div class="sidebar-section">
  <label class="sidebar-label">系统提示词</label>
  <textarea id="chatSystem" rows="3" placeholder="设定 AI 角色和行为……"></textarea>
</div>

<div class="sidebar-section">
  <label class="sidebar-label">温度: <span id="chatTempVal">0.7</span></label>
  <input type="range" id="chatTemp" min="0" max="2" step="0.1" value="0.7" />
</div>

<div class="sidebar-section">
  <label class="sidebar-label">最大 Token</label>
  <input type="number" id="chatMaxTokens" value="4096" min="1" max="65535" />
</div>

<div class="sidebar-section">
  <label class="sidebar-label" style="display:flex;align-items:center;gap:8px;">
    <input type="checkbox" id="chatThinking" />
    深度思考 (enable_thinking)
  </label>
</div>
```

---

## 三、流式输出实现

### 3.1 请求格式（纯文本）

```json
POST https://apihub.agnes-ai.com/v1/chat/completions
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "model": "agnes-2.0-flash",
  "messages": [
    { "role": "system", "content": "你是一个有帮助的助手。" },
    { "role": "user", "content": "上一轮问题" },
    { "role": "assistant", "content": "上一轮回答" },
    { "role": "user", "content": "当前问题" }
  ],
  "temperature": 0.7,
  "max_tokens": 4096,
  "stream": true,
  "chat_template_kwargs": {
    "enable_thinking": true
  }
}
```

### 3.1.1 请求格式（含图片 URL 输入）

Agnes-2.0-Flash **支持**通过图片 URL 进行图片理解。当消息包含图片时，`content` 使用数组格式：

```json
POST https://apihub.agnes-ai.com/v1/chat/completions
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "model": "agnes-2.0-flash",
  "messages": [
    {
      "role": "user",
      "content": [
        { "type": "text", "text": "请描述这张图片" },
        { "type": "image_url", "image_url": { "url": "https://example.com/image.jpg" } }
      ]
    }
  ],
  "temperature": 0.7,
  "max_tokens": 4096,
  "stream": true
}
```

### 3.2 SSE 响应解析（OpenAI 兼容格式）

```
data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","created":123,"model":"agnes-2.0-flash","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"Hello"},"finish_reason":null}]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" world"},"finish_reason":null}]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

### 3.3 流式处理逻辑

```
fetch(chatEndpoint, { body: JSON.stringify(req) })
  → response.body.getReader()
  → 逐行读取 → 按 "data: " 分割
  → JSON.parse(data) 
  → 提取 choices[0].delta.content
  → 追加到 AI 消息气泡的 textContent
  → 循环直到 "data: [DONE]"
```

### 3.4 消息气泡渲染

对话模式下 AI 消息实时流式追加：
```javascript
// 伪代码
let fullText = '';
const aiMsgBubble = createBubble('ai');

while (true) {
  const { done, chunk } = await reader.read();
  if (done) break;
  const lines = chunk.split('\n');
  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const data = line.slice(6);
      if (data === '[DONE]') break;
      const json = JSON.parse(data);
      const content = json.choices?.[0]?.delta?.content || '';
      fullText += content;
      aiMsgBubble.textContent = fullText;
      scrollToBottom();
    }
  }
}
```

---

## 四、多轮对话实现

### 4.1 消息历史构造（含图片 URL 支持）

```javascript
function buildChatMessages(conv, userPrompt, imageUrls) {
  const messages = [];

  // system prompt（如果设置了）
  const systemPrompt = document.getElementById('chatSystem').value.trim();
  if (systemPrompt) {
    messages.push({ role: 'system', content: systemPrompt });
  }

  // 历史对话（当前对话的所有 user/ai 消息对）
  for (const msg of conv.messages) {
    if (msg.role === 'user') {
      // 如果该用户消息有关联的图片 URL，使用 content 数组格式
      if (msg.refUrls && msg.refUrls.length > 0) {
        const content = [{ type: "text", text: msg.text }];
        for (const url of msg.refUrls) {
          content.push({ type: "image_url", image_url: { url: url } });
        }
        messages.push({ role: 'user', content: content });
      } else {
        messages.push({ role: 'user', content: msg.text });
      }
    } else if (msg.role === 'ai' && msg.isError !== true) {
      messages.push({ role: 'assistant', content: msg.text });
    }
  }

  // 当前用户消息（含图片 URL 时使用 content 数组）
  if (imageUrls && imageUrls.length > 0) {
    const content = [{ type: "text", text: userPrompt }];
    for (const url of imageUrls) {
      content.push({ type: "image_url", image_url: { url: url } });
    }
    messages.push({ role: 'user', content: content });
  } else {
    messages.push({ role: 'user', content: userPrompt });
  }

  return messages;
}
```

### 4.2 图片理解支持

Agnes-2.0-Flash **支持**通过图片 URL 进行图片理解（官方文档确认），因此：

- 对话模式下**保留**参考图 URL 输入框和缩略图显示
- 对话模式下 refUrls 数组会传入 `buildChatMessages()`，构建包含 `image_url` 类型的 content 数组
- 图片 URL 要求：必须公网可访问，建议使用 JPG/JPEG/PNG/WebP 格式
- 图片理解可与工具调用、流式输出和 Agent 工作流结合使用

> **注意**：如图片 URL 需要登录、鉴权或存在防盗链，模型可能无法读取。

---

## 五、对话气泡样式

### 5.1 不同于图片/视频模式的渲染

对话模式下 `buildBubbleContent()` 分支：

```javascript
if (chatMode === 'chat') {
  if (msg.role === 'user') {
    // 右对齐，蓝色背景气泡
    html += `<div class="chat-bubble chat-bubble-user">${msg.text}</div>`;
  } else {
    // 左对齐，灰色背景气泡，流式追加
    html += `<div class="chat-bubble chat-bubble-ai" id="${streamId}">${msg.text || ''}</div>`;
  }
}
```

### 5.2 CSS

```css
.chat-bubble {
  max-width: 80%; padding: 10px 14px;
  border-radius: var(--radius-sm); line-height: 1.6;
  font-size: 14px; word-break: break-word;
}
.chat-bubble-user {
  margin-left: auto; margin-right: 12px;
  background: var(--accent); color: #fff;
}
.chat-bubble-ai {
  margin-right: auto; margin-left: 12px;
  background: var(--bg-secondary); color: var(--text-primary);
}
```

---

## 六、发送与接收逻辑

### 6.1 `buildRequestBody()` 扩展

```javascript
function buildRequestBody(prompt, imageUrls) {
  const cMode = getChatMode();

  if (cMode === 'chat') {
    // 对话请求体
    const conv = getCurrentConv();
    const messages = buildChatMessages(conv, prompt, imageUrls);
    const body = {
      model: document.getElementById('chatModel').value,
      messages: messages,
      temperature: parseFloat(document.getElementById('chatTemp').value),
      max_tokens: parseInt(document.getElementById('chatMaxTokens').value),
      stream: true,
    };
    if (document.getElementById('chatThinking').checked) {
      body.chat_template_kwargs = { enable_thinking: true };
    }
    return body;
  }

  if (cMode === 'video') { /* 现有逻辑 */ }
  /* cMode === 'image' 现有逻辑 */
}
```

### 6.2 `sendMessage()` 扩展

```javascript
async function sendMessage() {
  // ...
  const cMode = getChatMode();
  
  if (cMode === 'chat') {
    const imageUrls = getRefUrls(); // 获取参考图 URL
    await sendChatMessage(prompt, imageUrls);
    return;
  }
  
  // 现有图片/视频逻辑
}
```

### 6.3 `sendChatMessage()` 新函数

```javascript
async function sendChatMessage(prompt, imageUrls) {
  const conv = getCurrentConv();
  const reqBody = buildRequestBody(prompt, imageUrls || []);
  const endpoint = 'https://apihub.agnes-ai.com/v1/chat/completions';
  
  // 先追加 user 消息（含图片 URL）
  addMessageToConv(conv.id, 'user', prompt, false, imageUrls || []);
  
  // 创建 AI 消息气泡（空内容，等待流式填充）
  const streamId = 'stream_' + Date.now();
  addMessageToConv(conv.id, 'ai', '', false, [], [], { _streamId: streamId });
  renderAllMessages();
  
  // 发起流式请求
  const response = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${apiKey}` },
    body: JSON.stringify(reqBody),
  });
  
  // SSE 流式读取
  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let fullText = '';
  let buffer = '';
  
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop(); // 保留不完整行
    for (const line of lines) {
      if (!line.startsWith('data: ')) continue;
      const data = line.slice(6);
      if (data === '[DONE]') continue;
      try {
        const json = JSON.parse(data);
        const content = json.choices?.[0]?.delta?.content || '';
        fullText += content;
        // 更新 DOM 中的气泡文字
        const el = document.getElementById(streamId);
        if (el) {
          el.textContent = fullText;
          scrollToBottom();
        }
      } catch {}
    }
  }
  
  // 流结束，保存完整文本到 IndexedDB
  const aiMsg = conv.messages.find(m => m.apiBodies?._streamId === streamId);
  if (aiMsg) {
    aiMsg.text = fullText;
    aiMsg.time = new Date().toISOString();
    await dbPut(conv);
  }
}
```

---

## 七、对话模式下的操作按钮

对话模式下消息气泡的操作按钮策略：

| 按钮 | user 消息 | AI 消息 |
|------|:---:|:---:|
| ❌ 删除 | 不显示 | 不显示 |
| 🔄 重生成 | 不显示 | **显示**（重新发送当前 prompt） |
| 📋 复制 | 显示 | **显示** |
| 请求体/响应体 | 不显示 | **显示**（折叠展开） |
| ⏸ 暂停自刷 | 不显示 | 不显示 |

### 重生成逻辑

对话模式重生成：
1. 保留当前 user 消息
2. 删除当前 AI 消息
3. 用相同的 messages[]（不含最后一条 assistant 回复）重新请求
4. 流式输出新回复

---

## 八、文件改动清单

| 区域 | 改动 |
|------|------|
| **HTML - 侧边栏** | 新增「对话配置」分组（chatModel, chatSystem, chatTemp, chatMaxTokens, chatThinking）|
| **HTML - 模式菜单** | 新增 💬 对话 选项 |
| **CSS** | 对话气泡样式、温度 slider 样式 |
| **DOM 引用** | 新增 chatModel, chatSystem, chatTemp, chatMaxTokens, chatThinking |
| **`init()`** | 恢复对话配置 localStorage |
| **`setChatMode()`** | 支持 'chat' 模式 |
| **`updateChatModeUI()`** | 对话模式保留 refUrl 输入框（因 agnes-2.0-flash 支持图片理解）|
| **`buildRequestBody()`** | 对话模式 → chat completions 格式 |
| **`sendMessage()`** | 对话模式 → 调用 `sendChatMessage()` |
| **新增 `sendChatMessage()`** | SSE 流式请求 + 打字机效果渲染 |
| **新增 `buildChatMessages()`** | 构造 messages[] 数组（system + 历史 + 当前）|
| **`buildBubbleContent()`** | 对话模式 → 气泡样式渲染 |
| **`regenerateFromUser()`** | 对话模式 → 删除 AI msg，重新流式请求 |
| **`getEndpoint()`** | 对话模式 → chat endpoint |

---

## 九、待确认 / 可扩展项

1. **图片理解 Prompt 优化**：Agnes-2.0-Flash 已支持图片 URL 输入和图片理解（通过 `image_url` 类型），后续可优化图片理解场景的 Prompt 模板（如截图分析、界面分析、物体识别等）
2. **工具调用 (Tool Use)**：API 支持 `tools` 参数，后续可加入智能体工作流
3. **Thinking 显示**：模型开启思考后，SSE 中可能有 `reasoning_content` 字段，可单独显示思考过程
4. **对话历史长度控制**：256K context window 很大，但多轮累积可能导致请求体过大，后续可加入消息条数裁剪逻辑
5. **中断生成**：添加停止按钮，用户在流式输出中可中断

---

## 十、代码审查发现的 3 个设计缺陷及修复方案

> 以下问题在代码探索阶段发现（2026-06-09），设计文档 v1.1 未覆盖，在此补充。

### 10.1 缺陷一：renderAllMessages 全量重渲染会打断流式输出

**问题**：当前 `renderAllMessages()` 执行 `$chatMessages.innerHTML = ''` 全量清空重建 DOM。
流式输出中 `sendChatMessage` 正在向某个 DOM 元素追加文本，如果此时触发 `renderAllMessages`
（如切换对话、重命名对话标题），流式内容全部丢失。

**修复方案**：

1. 流式 AI 消息在 `apiBodies` 中设置 `_streaming: true` 和 `_streamId` 标记
2. `buildBubbleContent()` 渲染时检测该标记，不用 `msg.text` 渲染，而是读取实时 DOM `textContent` 回填：

```javascript
// buildBubbleContent() 中 chat 模式的 AI 消息渲染
if (chatMode === 'chat' && msg.role === 'ai') {
  let text = msg.text || '';
  if (msg.apiBodies?._streaming && msg.apiBodies?._streamId) {
    const el = document.querySelector(`[data-stream-id="${msg.apiBodies._streamId}"]`);
    if (el) text = el.textContent || text;
  }
  html += `<div class="chat-bubble chat-bubble-ai" data-stream-id="${msg.apiBodies._streamId || ''}">${text}<span class="stream-cursor">|</span></div>`;
}
```

3. 流结束后 `_streaming = false`，后续渲染直接从 `msg.text` 读取（支持 Markdown 解析）

### 10.2 缺陷二：会话切换时应中断流式输出并提示用户

**问题**：用户在流式输出进行中切换对话，`switchConversation()` 直接切换 `currentConvId`，
原来对话的流式 SSE 连接仍在运行但没有接收端，造成资源浪费和状态混乱。

**修复方案**——`switchConversation()` 加拦截：

```javascript
async function switchConversation(id) {
  const currConv = getCurrentConv();
  if (currConv && convPendingRequests[currentConvId] > 0) {
    const confirmed = confirm(
      '当前对话正在进行中，切换可能导致图片/对话/视频显示错误，确定切换吗？'
    );
    if (!confirmed) return;
  }

  // 中断流式请求
  if (currentAbortController) {
    currentAbortController.abort();
    currentAbortController = null;
  }

  // 保存当前并切换
  if (currentConvId && getCurrentConv()) {
    await saveCurrentConv();
  }
  currentConvId = id;
  updateChatModeUI();
  // ... 后续逻辑
}
```

### 10.3 缺陷三：buildBubbleContent 中 chat 模式分支需要完整实现

**问题**：设计文档中气泡样式伪代码过于简略，未说明 chat 模式如何完整处理消息的所有 UI 元素。

**补充完整代码**：

```javascript
function buildBubbleContent(msg) {
  const cMode = getChatMode();

  // ========== Chat 模式 ==========
  if (cMode === 'chat') {
    let html = '';
    const streamId = msg.apiBodies?._streamId || '';
    const streaming = msg.apiBodies?._streaming === true;

    if (msg.role === 'user') {
      html += `<div class="chat-bubble chat-bubble-user">${escapeHtml(msg.text)}</div>`;
      html += getChatMsgActions(msg, 'user');
    } else {
      let text = msg.text || '';
      if (streaming && streamId) {
        const el = document.querySelector(`[data-stream-id="${streamId}"]`);
        if (el) text = el.textContent || text;
      }

      html += `<div class="chat-bubble chat-bubble-ai" data-stream-id="${streamId}">`;
      if (msg.isError) {
        html += `<div class="bubble-error">${escapeHtml(text)}</div>`;
      } else if (streaming) {
        html += `${escapeHtml(text)}<span class="stream-cursor">|</span>`;
      } else {
        html += text; // 流完成后支持 Markdown
      }
      html += '</div>';
      html += getChatMsgActions(msg, 'ai');
    }

    // 请求体/响应体折叠
    if (msg.apiBodies?.requestBody) {
      html += buildApiCollapse(msg);
    }
    return html;
  }

  // ========== 图片/视频模式：现有逻辑 ==========
  // ... 保持不变
}
```

**对话模式 vs 图片/视频模式 UI 差异**：

| 元素 | 图片/视频模式 | 对话模式 |
|------|:---:|:---:|
| 消息布局 | 左头像 + 右气泡框 | 纯气泡，无头像 |
| 用户消息 | 普通文本 div（可点击编辑）| 右对齐蓝色气泡 |
| AI 消息 | 普通文本 div + 图片/视频网格 | 左对齐灰色气泡 + Markdown |
| 参考图 | 显示缩略图行 | 不显示 |
| 生成图 | gen-image-wrapper | 不适用 |
| 重生成按钮 | 用户消息显示 | **AI 消息显示** |
| 复制按钮 | 用户消息显示 | 用户和 AI **都显示** |
| 删除按钮 | 用户消息显示 | **不显示** |
| 请求体折叠 | 用户+AI | 用户+AI |

---

*文档版本：v1.3*
*日期：2026-06-09*
*变更 v1.2：补充代码审查发现的 3 个设计缺陷及修复方案（renderAllMessages 流式保护、会话切换中断保护、buildBubbleContent chat 分支具体化）*
*变更 v1.3：新增 Thinking 内容展示（reasoning_content 解析、展开/折叠功能）和 Markdown 渲染支持（引入 marked.js）*

---

## 十一、Thinking 内容展示（v1.3 新增）

### 11.1 SSE 中的 thinking 字段

Agnes-2.0-Flash 在深度思考模式下，SSE 流式响应中会包含 `reasoning_content` 字段：

```json
{
  "choices": [{
    "delta": {
      "content": "...",
      "reasoning_content": "正在思考..."
    }
  }]
}
```

### 11.2 实现方案

1. **流式读取时捕获**：`sendChatMessage` 中同时提取 `delta.content` 和 `delta.reasoning_content`
2. **实时预览**：thinking 内容实时追加到 DOM（`data-thinking-id`）
3. **持久化存储**：流结束后保存到 `msg.apiBodies._thinkingText`
4. **折叠/展开 UI**：点击头部切换显示隐藏

### 11.3 CSS 样式

```css
.thinking-block {
  margin-bottom: 10px; border-radius: var(--radius-sm);
  background: rgba(78,168,222,0.08); border: 1px solid rgba(78,168,222,0.2);
}
.thinking-header {
  display: flex; align-items: center; gap: 6px;
  padding: 6px 10px; cursor: pointer; font-size: 12px; color: var(--accent);
}
.thinking-content {
  display: none; padding: 8px 12px 10px 32px;
  font-size: 12px; line-height: 1.6; color: var(--text-secondary);
  border-top: 1px solid rgba(78,168,222,0.1); max-height: 300px; overflow-y: auto;
}
.thinking-content.open { display: block; }
```

---

## 十二、Markdown 渲染（v1.3 新增）

### 12.1 引入 marked.js

使用 CDN 引入轻量级 Markdown 解析库：

```html
<script src="https://cdn.jsdelivr.net/npm/marked/marked.min.js"></script>
```

配置：

```javascript
marked.setOptions({
  breaks: true,   // 换行转换为 <br>
  gfm: true,      // GitHub 风格 Markdown
});
```

### 12.2 渲染时机

- **流式输出中**：不使用 Markdown（纯文本 + 转义）
- **流结束后**：使用 `marked.parse(text)` 渲染完整回复

### 12.3 支持的 Markdown 语法

| 语法 | 渲染结果 |
|------|---------|
| `# H1` / `## H2` | 标题 |
| `**粗体**` / `*斜体*` | 格式化文本 |
| `列表` / `1. 有序列表` | HTML 列表 |
| `` `行内代码` `` | 代码片段 |
| ```` ```代码块``` ```` | 代码块 |
| `> 引用` | 引用块 |
| `[链接](url)` | 超链接 |
| `---` | 分割线 |
| `表格` | HTML 表格 |

### 12.4 Markdown 内容样式

```css
.md-content h1, .md-content h2, .md-content h3 { color: var(--text-primary); }
.md-content code { background: rgba(78,168,222,0.1); padding: 2px 5px; border-radius: 3px; }
.md-content pre { background: #0d1117; border: 1px solid var(--border); padding: 10px 12px; }
.md-content blockquote { border-left: 3px solid var(--accent); background: rgba(78,168,222,0.05); }
```
