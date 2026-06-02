# 移动端 Claude 客户端 — 编码摘要

> 对应设计文档：[移动端 Claude 客户端-current.md](移动端%20Claude%20客户端-current.md) · [API](移动端%20Claude%20客户端-api-current.md)
> 本文档只回答"每个方法怎么写"，注释一律中文（仅符号保留英文）。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-06-01 | 初始版本 |
| current | 2026-06-02 | 补双链路断线恢复方法：sidecar 后端自动重连+resume、惰性 ensureSessionResumable、未决弹窗重投/离线推送 |

---

## 1. 核心业务规则

- 会话与浏览器连接解耦：断连不杀会话，sidecar 持续跑。
- 权限默认拒绝：`canUseTool` 超时/会话中断一律按 deny。
- 事件单调 `seq`：断连重连按 `seq` 回放，无丢无重。
- 不复制 transcript：全量历史唯一权威是 `~/.claude/projects/*.jsonl`，SQLite 只存元数据。
- 通知只在无活跃前台连接时发；前台在看则跳过；离线时若有未决权限/提问也推送提醒。
- Node sidecar 只绑 `127.0.0.1`，不对外暴露。
- sidecar(WS②) 断开后端自动重连并 resume 全部会话，无需手动重进；断开期间不静默丢消息（send/attach 前惰性恢复）。
- 未决权限/提问请求：断线重连按 `seq` 回放 + 重投，决策到达或本轮结束才清除。

---

## 2. 接口入口指针

> 字段级契约见 api 文档，本节只列入口。

| 接口 | 实现类 #方法 |
|------|-------------|
| `WS /api/claude-chat/ws` | `ClaudeChatWebSocketHandler#handleTextMessage` |
| `GET /api/claude-chat/sessions` | `ClaudeChatSessionController#list` |
| `DELETE /api/claude-chat/sessions/{id}` | `ClaudeChatSessionController#delete` |

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.toolbox.claudechat.config.ClaudeChatToolDescriptor` | 新建 | ToolDescriptor 实现，id=claude-chat |
| `com.exceptioncoder.toolbox.claudechat.config.ClaudeChatWebSocketConfig` | 新建 | 注册 handler 到 `/api/claude-chat/ws`，复用 webterm 配置 |
| `com.exceptioncoder.toolbox.claudechat.config.ClaudeChatWebSocketHandler` | 新建 | 浏览器侧 WS，收发 JSON 转交 Service |
| `com.exceptioncoder.toolbox.claudechat.service.ClaudeChatService` | 新建 | 会话编排、事件缓冲、通知判定 |
| `com.exceptioncoder.toolbox.claudechat.service.SidecarClient` | 新建 | Java↔Node WS 客户端 |
| `com.exceptioncoder.toolbox.claudechat.service.SidecarProcessRegistry` | 新建 | Node 进程生命周期 |
| `com.exceptioncoder.toolbox.claudechat.service.NotificationService` | 新建 | 完成通知编排，按配置选渠道，读 feature-config |
| `com.exceptioncoder.toolbox.claudechat.service.notify.NotificationSender` | 新建 | 推送渠道接口 |
| `com.exceptioncoder.toolbox.claudechat.service.notify.BarkSender` | 新建 | iPhone Bark 实现（GET 推送） |
| `com.exceptioncoder.toolbox.claudechat.service.notify.NtfySender` | 新建 | Android ntfy 实现（POST 推送） |
| `com.exceptioncoder.toolbox.claudechat.repository.ClaudeChatSessionRepository` | 新建 | 会话元数据 JDBC |
| `com.exceptioncoder.toolbox.claudechat.domain.ClaudeChatSession` | 新建 | 会话实体 |
| `com.exceptioncoder.toolbox.claudechat.domain.SessionStatus` | 新建 | 枚举 RUNNING/IDLE/INTERRUPTED/DONE |
| `com.exceptioncoder.toolbox.claudechat.api.ClaudeChatSessionController` | 新建 | 会话列表/删除 + 磁盘历史列表 GET /history |
| `com.exceptioncoder.toolbox.claudechat.service.SessionHistoryService` | 新建 | 扫描 ~/.claude/projects 解析历史会话 |
| `com.exceptioncoder.toolbox.claudechat.api.dto.HistorySessionView` | 新建 | 历史会话视图(sdkSessionId/cwd/title/lastModified/messageCount) |
| `sidecar/claude-agent/src/server.ts` | 新建 | WS server + 端口握手 |
| `sidecar/claude-agent/src/sessionManager.ts` | 新建 | 多会话 query() 路由 |
| `sidecar/claude-agent/src/permissions.ts` | 新建 | canUseTool 回调 |
| `frontend/src/features/claude-chat/*` | 新建 | 见设计文档 §6 |

### 关键方法签名与职责

```
// Java
ClaudeChatService#openSession(String connId, OpenReq req): void — 拉起 sidecar、建会话、持久化、回 ready
ClaudeChatService#attach(String connId, String sessionId, long lastSeq): void — 回放 seq>lastSeq 的缓冲并续流
ClaudeChatService#switchSession(String connId, String sessionId): void — 取 sdkSessionId 触发 sidecar resume
ClaudeChatService#resumeHistory(ws, sdkSessionId, cwd): void — 为磁盘历史会话建元数据行后 resume
SessionHistoryService#list(String cwd): List<HistorySessionView> — 编码 cwd→目录(大小写不敏感合并)→扫 *.jsonl 解析
SessionHistoryService#encode(String cwd): String — 非字母数字字符全替换为 '-'
SessionHistoryService#parseTitle(Path jsonl): String — 逐行读，命中首条 user 文本即停，截断 ~60 字
ClaudeChatService#onUserMessage(String sessionId, String text): void — 透传给 sidecar
ClaudeChatService#onDecision(String sessionId, Decision d): void — 回灌权限/提问决策给 sidecar
ClaudeChatService#onSidecarEvent(String sessionId, ServerEvent e): void — 打 seq、入缓冲、下发浏览器、result 触发通知
SidecarProcessRegistry#ensureSidecar(): SidecarHandle — 未运行则 ProcessBuilder 拉起并握手取端口
SidecarProcessRegistry#onExit(): void — 标记挂靠会话为 INTERRUPTED

// 断线恢复（双链路韧性）
ClaudeChatService#onSidecarDown(): void — WS② 断开：会话标 INTERRUPTED + emit SIDECAR_DOWN + 触发 scheduleSidecarRecovery
ClaudeChatService#scheduleSidecarRecovery(): void — AtomicBoolean 去重；虚拟线程退避循环 ensureStarted+ensureConnected，成功后 resumeAllSessions
ClaudeChatService#resumeAllSessions(): void — 对每个有 sdkSessionId 的会话 resumeSession + 置 IDLE + emit Ready（前端清错恢复）
ClaudeChatService#ensureSessionResumable(SessionCtx ctx): boolean — sidecar 不在线则就地 ensureStarted+ensureConnected+resume；attach/sendUserMessage 前调，失败回 SIDECAR_DOWN
ClaudeChatService#onDecisionPrompt(SessionCtx ctx, ServerMessage msg, String title, String body): void — 记 ctx.pendingRequest；无活跃前台时 notify 推送提醒
ClaudeChatService#redeliverPending(SessionCtx ctx, long lastSeq): void — attach 重连后重投未被 seq 覆盖的未决权限/提问弹窗
SidecarClient#startSession(String sessionId, String cwd): void
SidecarClient#resumeSession(String sessionId, String sdkSessionId): void
NotificationService#notifyDone(ClaudeChatSession s, ResultEvent r): void — 读 feature-config，遍历启用渠道分发
NotificationSender#send(String title, String body): void — 渠道推送接口
BarkSender#send(...): void — GET {barkBaseUrl}/{deviceKey}/{title}/{body}
NtfySender#send(...): void — POST {ntfyBaseUrl}/{topic}，Title header + body

// Node (TS)
sessionManager.startSession(sessionId, cwd): 启动 query() 流，转发 SDK 消息为 ServerEvent
sessionManager.resumeSession(sessionId, sdkSessionId): query({options:{resume}})
permissions.canUseTool(toolName, input, ctx): Promise<Allow|Deny> — 发 request、阻塞等 decision、超时 deny
```

---

## 4. 数据结构

### 关键表及字段

```
表名：claude_chat_session
  id            TEXT PRIMARY KEY,
  cwd           TEXT NOT NULL,
  title         TEXT,
  sdk_session_id TEXT,                  -- SDK 侧 session_id，resume 用
  status        TEXT NOT NULL,          -- RUNNING/IDLE/INTERRUPTED/DONE
  started_at    INTEGER NOT NULL,
  last_seen_at  INTEGER NOT NULL
索引：CREATE INDEX IF NOT EXISTS idx_ccs_last_seen ON claude_chat_session(last_seen_at)
约束：所有 DDL 必须 CREATE TABLE/INDEX IF NOT EXISTS（SchemaInitializer 每次启动重跑）
```

### 关键 DTO 字段

```
// WS 消息见 api 文档 §2；前后端各维护一份 type 定义
// ClaudeChatSessionView: id, cwd, title, sdkSessionId, status, startedAt, lastSeenAt, live(boolean)
```

---

## 5. 重要约束与边界

- 事件缓冲：内存环形，仅缓当前会话当前一轮；历史轮次回放走读 JSONL。
- 并发：一个 Node 进程内多 `query()` 按 sessionId 路由；Java 侧 connId↔sessionId 多对一。
- sidecar 端口：`127.0.0.1` 固定可配端口（`toolbox.claude-chat.sidecar-port` 默认 18890），Java 经环境变量 `CLAUDE_CHAT_SIDECAR_PORT` 传给 sidecar。
- 不处理：多用户、鉴权、Web Push 订阅、IDE 诊断/diff 能力。

---

## 6. 下游依赖调用

```
// Java → Node sidecar：WebSocket（SidecarClient），非 HTTP
// Java → 推送：java.net.http.HttpClient → Bark(GET)/ntfy(POST)，渠道与参数取自 feature-config
//   feature-config 键：claude-chat.notify.bark.{enabled,baseUrl,deviceKey}
//                      claude-chat.notify.ntfy.{enabled,baseUrl,topic}
// Node → Claude：@anthropic-ai/claude-agent-sdk query()/canUseTool/options.resume
// feature-config 读取：复用「feature-config 通用配置存储」既有 Service（键如 claude-chat.notify.*）
```

---

## 7. 异常处理要点

- sidecar 未就绪/崩溃 → 下发 `error{code:SIDECAR_DOWN}` + 会话标 INTERRUPTED；随后后端自动重连+resume 恢复（emit Ready 清错），无需手动重进会话。
- attach/switch 会话不存在 → `error{code:SESSION_NOT_FOUND}`。
- 权限请求超时 → 按 deny resolve，发 `error{code:PERMISSION_TIMEOUT}`（非致命，仅提示）。
- WS 断开 → 不杀会话，保留事件缓冲等待 attach。
- `mvn package` 需保证新模块不破坏 fat-jar 前端嵌入流程。
