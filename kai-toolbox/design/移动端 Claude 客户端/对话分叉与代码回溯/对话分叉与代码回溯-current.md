# 对话分叉与代码回溯（fork / rewind）— 技术设计

> 隶属「移动端 Claude 客户端」。复刻官方 VSCode 插件的 rewind 菜单：从某条用户消息「分叉对话 / 回退代码 / 两者」。
> 最后更新：2026-06-03

## 1. 目标与边界

- **目标**：用户消息上提供 rewind 菜单 3 项——
  1. **Fork conversation from here**：截到该消息分叉出新会话续跑（旧会话保留）。
  2. **Rewind code to here**：把文件回退到该消息时的快照。
  3. **两者**。
- **不做**：不删原会话；不做可视化 diff 预览（仅返回受影响文件数）。
- **安全前提**：开发版改造，但**后端/sidecar 与稳定版共用**——后端只做加法 + checkpointing 设可配置开关，确保稳定版不受影响、可作远程救急后路。

## 2. SDK 能力（已确认）

| 能力 | API | 约束 |
|------|-----|------|
| 分叉对话 | `forkSession(sessionId, { upToMessageId })`（顶层函数，操作 transcript 文件） | **不需要活着的 query、不需要 checkpointing**；返回新 sessionId，再 resume |
| 回退代码 | `query.rewindFiles(userMessageId)` | **需要活着的 query + `options.enableFileCheckpointing=true`** |

两者都需要**那条用户消息的 SDK transcript uuid**。

## 3. 关键设计点：怎么拿用户消息的 SDK uuid（核心难点）

现状：前端用户消息用自生成 `iXXX` id，**没存 SDK uuid**。方案：

- **首选**：sidecar 在 SDK 流里捕获用户消息的 `uuid`（每个 SDKMessage 带 uuid/parentUuid）。runTurn 发 prompt 后，从流中识别本轮 user 消息的 uuid，`emit { type:'userMessage', uuid }`，Java 透传，前端把它挂到刚发出的那条 user `ChatItem` 上。
- **兜底**（若流里取不到 user 文本消息的 uuid）：本轮 `result` 后读 transcript jsonl 末尾的 user 消息 uuid。
- **历史消息**：`loadMessages` 读 transcript 时本就有每条的 uuid，一并带出挂到 ChatItem。

> 这是唯一需要在你机器上实测确认的点（我无法在此跑 SDK）。先按「流里捕获」实现，跑不通切兜底。

## 4. 分阶段（降风险）

| 阶段 | 内容 | 风险 | 对稳定版后端影响 |
|------|------|------|------------------|
| **P1** | Fork conversation from here | 低：`forkSession` 是文件级操作，**不开 checkpointing、不需活 query** | 仅新增「捕获 uuid」事件（加法），无行为变更 |
| **P2** | Rewind code to here（+两者） | 中：需开 `enableFileCheckpointing`（影响所有会话）+ `rewindFiles` 控制调用 | checkpointing 设 `toolbox.claude-chat.file-checkpointing`（可 env 关闭）兜底 |

先做 P1（最有用、最安全），P2 单独再上。

## 5. 接口契约（WS 消息，全加法）

| 消息 | 方向 | 内容 | 阶段 |
|------|------|------|------|
| `userMessage` | sidecar→Java→前端 | `{ uuid }`（关联刚发出的 user 消息） | P1 |
| `forkSession` | 前端→Java→sidecar | `{ upToMessageId }` → forkSession → 返回新 sessionId（前端 switchTo 续跑） | P1 |
| `rewindCode` | 前端→Java→sidecar | `{ userMessageId }` → rewindFiles | P2 |
| `rewindResult` | sidecar→Java→前端 | `{ ok, changedFiles, error? }` | P2 |

前端 `ChatItem(user)` 增 `sdkUuid?: string`；rewind 菜单仅在有 uuid 时可用。

## 6. 编码落点

```text
sidecar/claude-agent/src/sessionManager.ts   [修改] 捕获 user uuid emit；fork/rewindFiles 命令；P2 query 选项 enableFileCheckpointing
sidecar/claude-agent/src/server.ts            [修改] dispatch forkSession / rewindCode
tools/.../api/dto/ClientMessage.java          [修改] ForkSession / RewindCode
tools/.../api/dto/ServerMessage.java          [修改] UserMessage / RewindResult
tools/.../config/ClaudeChatWebSocketHandler.java [修改] 路由
tools/.../service/SidecarClient.java          [修改] forkSession / rewindCode 发送
tools/.../service/ClaudeChatService.java       [修改] 透传 + fork 后建元数据行/切会话
tools/.../config/ClaudeChatProperties.java     [修改] P2 file-checkpointing 开关（默认 true，可 env 关）
frontend/.../types.ts、useClaudeChatSocket.ts  [修改] sdkUuid、forkSession/rewindCode、事件处理
frontend/.../components/MessageList.tsx        [修改] user 消息上的 rewind 菜单
frontend/.../api.ts                            [修改] loadMessages 带出每条 sdkUuid
```

## 6.1 P1 实现落地（2026-06-03）

已全量编码并通过编译（sidecar `tsc` / Java `compile` / 前端 claude-chat 改动类型干净）：

- **sidecar** `sessionManager.ts`：`handle()` 的 `case 'user'` 捕获 `m.uuid`（非 tool_result、非合成）→ `emitSelf({type:'userMessage', uuid})`；新增 `SessionManager.forkSession(id, upToMessageId)` 调顶层 `forkSession(sdkSessionId, {upToMessageId, dir:cwd})` → emit `{type:'forked', sdkSessionId, cwd}`。`server.ts` dispatch `forkSession`。
- **Java**：`ClientMessage.ForkSession(upToMessageId)`；`ServerMessage.UserMessage(seq,uuid)` / `Forked(seq,sessionId)`；`SidecarClient.forkSession`；`ClaudeChatService.forkSession()` + `onSidecarEvent` 新增 `userMessage`/`forked` 分支，`onForked()` 用新 sdkSessionId 建一条 `claude_chat_session` 行（同 resumeHistory）再回 `Forked` 让前端切。
- **前端**：`ChatItem(user).sdkUuid`；hook `applyEvent` 处理 `userMessage`（挂到最近一条未标记 user 项）/`forked`（自动 `switchTo` 新会话）；`forkSession()` 发送器；`MessageList` 在带 `sdkUuid` 的用户气泡下显示「从此处分叉」按钮，`ChatPage` 传 `onFork={chat.forkSession}`。

**全程加法**：未改动任何既有方法逻辑；稳定版（共用后端+sidecar）只会额外收到 `userMessage`/`forked` 事件，其 switch 不匹配即忽略，零行为变更。

**待你机器实测确认**：`userMessage` 的 uuid 捕获是否触发、是否对应到刚发出的那条用户消息（我无法本地跑 SDK）。若 live 流里取不到 user 文本消息 uuid，按 §3 兜底（result 后读 transcript 末尾 user uuid）。验证步骤：编译重启后发一条消息，看该气泡下是否出现「从此处分叉」按钮；点它应切到一个截断到该处的新会话。

## 7. 风险与待确认

| 风险 | 处理 |
|------|------|
| 取不到 user 消息 uuid | 兜底读 transcript；P1 先验证 |
| checkpointing 影响稳定版会话 | 配置开关，可 env 关闭；P2 才引入 |
| fork 后新会话续跑的元数据/列表 | fork 返回新 sessionId → 建一条 claude_chat_session 行（同 resumeHistory），前端 switchTo |
| 无法本地编译/跑 SDK 验证 | 你机器实测，按需迭代 |
