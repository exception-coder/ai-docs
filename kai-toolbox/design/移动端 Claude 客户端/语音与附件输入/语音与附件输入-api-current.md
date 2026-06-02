# 语音与附件输入 — 接口契约

> 配套 [语音与附件输入-current.md](语音与附件输入-current.md)。本期新增 2 个 HTTP 端点 + 扩展 1 个 WS 消息。
> 所有端点仅本机（localhost / tunnel），无鉴权，沿用 kai-toolbox 单用户定位。
> 最后更新日期：2026-06-01

---

## 1. HTTP：语音转写

### `POST /api/claude-chat/stt`

录音音频转文字（同步返回）。

**请求**

| 项 | 值 |
|----|----|
| Content-Type | 上传音频的真实 MIME（如 `audio/webm`、`audio/mp4`、`audio/wav`） |
| Body | 原始音频字节（raw body，非 multipart） |
| Query `language`（可选） | ISO 639-1 码或 `auto`，默认 `auto` |

**响应 200**

```json
{ "text": "转写后的纯文本" }
```

**错误**

| HTTP | code | 场景 |
|------|------|------|
| 503 | `ASR_UNAVAILABLE` | faster-whisper 服务未启动 / 探活失败 |
| 400 | `EMPTY_AUDIO` | 上传为空或时长 0 |
| 500 | `TRANSCRIBE_FAILED` | ffmpeg 转码或转写过程异常 |

> 实现：`SttController#transcribe` → `SttService`（ffmpeg 转 16kHz mono wav → `SpeechToTextClient` 调 `/asr` → 合并 segment 取纯文本）。

---

## 2. HTTP：附件上传

### `POST /api/claude-chat/sessions/{sessionId}/attachments`

上传单个附件，落盘到会话 cwd 专用目录，返回句柄。

**请求**

| 项 | 值 |
|----|----|
| Path `sessionId` | 目标会话 id（决定落盘目录） |
| Content-Type | `multipart/form-data` |
| Part `file` | 文件二进制 |

**响应 200**

```json
{
  "id": "att_xxxxxxxx",
  "name": "screenshot.png",
  "mime": "image/png",
  "size": 20480,
  "path": "D:/proj/.kai-chat-attachments/<sid>/1730-screenshot.png"
}
```

| 字段 | 说明 |
|------|------|
| `id` | 附件句柄 id，前端用于 chip 增删 |
| `name` | 原始文件名（已消毒） |
| `mime` | 探测到的 MIME |
| `size` | 字节数 |
| `path` | 服务端绝对路径，**前端只透传给 send，不展示** |

**错误**

| HTTP | code | 场景 |
|------|------|------|
| 404 | `SESSION_NOT_FOUND` | sessionId 不存在 |
| 413 | `FILE_TOO_LARGE` | 超过单文件上限（默认 50MB） |
| 415 | `UNSUPPORTED_TYPE` | 类型不在白名单（如可执行文件） |

> 实现：`AttachmentController` → `AttachmentStorageService#store`。

---

## 3. WS 协议扩展：`send` 带附件

复用既有 `/api/claude-chat/ws`，仅扩展 `send` 消息（向前兼容：不带 `attachments` 时行为不变）。

**客户端 → 服务端**

```ts
// types.ts: ClientMessage 的 send 分支
| { type: 'send'; text: string; attachments?: Attachment[] }

interface Attachment {
  name: string   // 原始文件名（展示用）
  path: string   // 上传响应里的服务端绝对路径
}
```

```java
// ClientMessage.java
record Send(String text, List<Attachment> attachments) implements ClientMessage {
    record Attachment(String name, String path) {}
}
```

**服务端处理**：`ClaudeChatService#sendUserMessage` 把 `attachments` 路径拼成结构化提示追加到 `text` 末尾，再以原 `userMessage`（仅 `text`）转发给 sidecar——**sidecar 协议、`SidecarClient.userMessage` 签名均不变**。

拼接示例：

```
<用户原文>

[附件] 用户上传了以下文件，需要时请用 Read 工具查看：
- screenshot.png → D:/proj/.kai-chat-attachments/<sid>/1730-screenshot.png
- spec.pdf → D:/proj/.kai-chat-attachments/<sid>/1730-spec.pdf
```

> 服务端事件（`ServerMessage`）无需改动，附件读取过程以既有 `toolUse(Read)` / `toolResult` 事件自然呈现。
