# 语音输入与附件发送 — 接口契约

> 配套 [语音输入与附件发送-current.md](语音输入与附件发送-current.md)。仅记录本子模块**新增/变更**的对外契约；父需求既有的 WS 消息见父需求 api 文档。
> 最后更新日期：2026-06-01

---

## 1. REST：附件上传

### 1.1 上传附件

`POST /api/claude-chat/sessions/{sessionId}/attachments`

- **Content-Type**：`multipart/form-data`
- **路径参数**：`sessionId` — 目标会话 id（决定落盘的 cwd 与附件目录）
- **表单字段**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | file | 是 | 单个附件文件，可多次调用上传多个 |

- **成功响应 `200`**：`AttachmentRef`

```json
{
  "id": "a1b2c3",
  "name": "screenshot.png",
  "path": "D:/Users/zhang/IdeaProjects/kai-toolbox/.claude-chat-attachments/<sessionId>/a1b2c3-screenshot.png",
  "type": "image/png",
  "size": 20480
}
```

- **错误响应**：

| HTTP | code | 场景 |
|------|------|------|
| 404 | `SESSION_NOT_FOUND` | sessionId 不存在 |
| 413 | `ATTACHMENT_TOO_LARGE` | 超过单文件上限 |
| 415 | `ATTACHMENT_TYPE_NOT_ALLOWED` | 不在类型白名单 |
| 400 | `BAD_FILENAME` | 文件名非法 / 路径穿越 |

### 1.2 字段约束

| 字段 | 约束 |
|------|------|
| `name` | 原始文件名，保留扩展名 |
| `path` | 服务端绝对路径，位于 `{会话cwd}/.claude-chat-attachments/{sessionId}/` 内 |
| `type` | MIME 类型，前端据此渲染缩略图/文件图标 |
| `size` | 字节数 |

> 白名单（默认，可配 `toolbox.claude-chat.attachment.allowed-types`）：`image/*`、`application/pdf`、`text/*`、常见代码/文本扩展名。
> 单文件上限默认 20MB（`toolbox.claude-chat.attachment.max-size`）。

---

## 2. WebSocket：`send` 消息扩展

端点不变：`/api/claude-chat/ws`。仅 `send` 消息体新增可选 `attachments`。

### 2.1 客户端 → 服务端：`send`（变更）

```jsonc
{
  "type": "send",
  "text": "帮我看下这张报错截图和这个日志文件",
  "attachments": [                       // 新增，可选；无附件时省略或传 []
    {
      "id": "a1b2c3",
      "name": "screenshot.png",
      "path": "D:/.../.claude-chat-attachments/<sessionId>/a1b2c3-screenshot.png",
      "type": "image/png",
      "size": 20480
    }
  ]
}
```

- `attachments` 中每项为 §1.1 上传返回的 `AttachmentRef`（前端原样回传）。
- 服务端只信任 `path`，并二次校验其仍位于该会话附件目录内。

### 2.2 服务端 → sidecar（内部协议，方案A）

> 非对外契约，记录 Java↔Node 的内部变化，便于实现对齐。

- **方案A（推荐）**：`SidecarClient.userMessage` 把附件路径拼进文本，sidecar 协议不变：

```jsonc
{ "type": "user", "sessionId": "...", "text": "原文本\n\n[附件]\n- D:/.../a1b2c3-screenshot.png\n- D:/.../log.txt" }
```

- **方案B（备选）**：新增结构化字段，sidecar 解析为多模态 content blocks：

```jsonc
{ "type": "user", "sessionId": "...", "text": "原文本",
  "attachments": [{ "path": "...", "type": "image/png" }] }
```

### 2.3 服务端 → 客户端

无新增事件。附件被 Claude 读取后，照常通过既有 `toolUse`（Read）/`assistantDelta` 等事件体现，无需新增 ServerMessage 类型。
