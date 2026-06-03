# 移动端 Claude 客户端 — 接口契约（WebSocket 协议）

> 对外接口以 WebSocket 为主，REST 仅会话列表/删除。最后更新：2026-06-03。
> 设计见 [移动端 Claude 客户端-current.md](移动端%20Claude%20客户端-current.md)。

## 1. 接口清单

| 通道 | 路径 | 说明 |
|------|------|------|
| WebSocket | `/api/claude-chat/ws` | 浏览器 ↔ Java，双向 JSON 消息 |
| REST GET | `/api/claude-chat/sessions` | 列出工具内会话（SQLite 元数据） |
| REST DELETE | `/api/claude-chat/sessions/{id}` | 删除会话元数据 |
| REST GET | `/api/claude-chat/history?cwd=...` | 列出某 cwd 磁盘上的 Claude Code 历史会话 |

> 内部 Java ↔ Node sidecar 的 WS 协议与本表浏览器侧消息**语义同构**，差异仅在 sidecar 多了 `sdkSessionId` 透传；不作为对外契约单列。

## 2. WebSocket 消息契约

### 2.1 客户端 → 服务端（ClientMessage）

| type | 字段 | 说明 |
|------|------|------|
| `open` | `cwd`, `model?`, `mode?` | 新建会话（`mode` 为初始权限模式，缺省 `default`） |
| `attach` | `sessionId`, `lastEventSeq` | 重连进行中的会话，请求回放 |
| `switchSession` | `sessionId` | 切到工具内会话（触发 resume） |
| `resumeHistory` | `sdkSessionId`, `cwd` | 续跑磁盘上的某历史会话（建元数据行后 resume） |
| `send` | `text`, `attachments?` | 下发一条用户消息（`attachments` 为已上传文件引用 `{name,path}`） |
| `decision` | `reqId`, `behavior`(`allow`/`deny`), `updatedInput?`, `answers?` | 回灌权限/提问决策 |
| `interrupt` | — | 中断当前轮 |
| `setMode` | `mode`(`default`/`acceptEdits`/`plan`/`bypassPermissions`) | 切换会话权限模式（下一轮生效） |
| `setModel` | `model` | 切换会话模型（下一轮生效；取值来自 `models` 事件） |
| `forkSession` | `upToMessageId` | 从某条用户消息分叉出新会话（旧会话保留）；`upToMessageId` 为该消息 SDK transcript uuid |

### 2.2 服务端 → 客户端（ServerMessage，均带 `seq`）

| type | 字段 | 说明 |
|------|------|------|
| `ready` | `sessionId`, `sdkSessionId`, `slashCommands?`, `status?` | 会话就绪；`slashCommands` 来自 SDK init；`status` 供前端同步 `running` 纠偏 |
| `assistantDelta` | `text` | assistant 文本增量 |
| `toolUse` | `toolName`, `input` | 工具调用开始（渲染气泡） |
| `toolResult` | `toolName`, `output`(摘要), `isError?` | 工具结果 |
| `permissionRequest` | `reqId`, `toolName`, `input` | 权限弹窗请求 |
| `questionRequest` | `reqId`, `questions[]` | AskUserQuestion 弹窗请求 |
| `decisionResolved` | `reqId` | 某权限/提问已被某端处理，通知其它端关闭同一弹窗（多端同看） |
| `models` | `models[]`(`{value,displayName,description}`), `current` | 会话可用模型清单（SDK `supportedModels`）+ 当前模型 |
| `userMessage` | `uuid` | 关联刚发出的用户消息与其 SDK transcript uuid（供「从此处分叉」定位） |
| `forked` | `sessionId` | 分叉完成，`sessionId` 为新建工具内会话 id（前端 `switchTo` 续跑） |
| `replayGap` | `missingFrom`, `missingTo` | 重连回放空洞（缺失事件已被有界缓冲淘汰）；`seq` 固定 0、仅发该连接、不入缓冲 |
| `result` | `usage?`, `stopReason` | 本轮结束（触发完成通知判定） |
| `error` | `code`, `message` | 错误（如 `SIDECAR_DOWN`/`SESSION_NOT_FOUND`/`FORK_FAILED`） |

### 2.3 关键结构

```
questions[]: { question, header, options: [{label, description}], multiSelect }
decision.answers: { [question]: string | string[] }   // 对应 AskUserQuestion 回填
permissionRequest.input / decision.updatedInput: 工具原始参数对象（allow 时可改）
```

## 3. REST 响应

```
GET /sessions → ClaudeChatSessionView[]
ClaudeChatSessionView: { id, cwd, title, sdkSessionId, status, startedAt, lastSeenAt, live }
// live=true 表示该会话当前仍挂在活跃 sidecar 上，可 attach

GET /history?cwd=<工作目录>（cwd 可选）→ HistorySessionView[]
HistorySessionView: { sdkSessionId, cwd, title, lastModified, messageCount }
// cwd 指定 → 该目录(大小写不敏感合并变体)；cwd 留空 → 跨 ~/.claude/projects 所有目录列最近 N 条
// 每条 cwd 为从 jsonl 解析出的真实工作目录；按 lastModified 倒序；按 sdkSessionId 去重
```

## 4. 错误码

| code | 含义 |
|------|------|
| `SIDECAR_DOWN` | Node sidecar 未就绪/已崩溃，可重试 |
| `SESSION_NOT_FOUND` | attach/switch 的会话不存在 |
| `PERMISSION_TIMEOUT` | 权限请求超时已按 deny 处理（仅通知，非致命） |
| `FORK_FAILED` | 分叉失败（会话未就绪或 SDK forkSession 报错），非致命 |
