# 权限模式切换 — 编码摘要

> 配套 [权限模式切换-current.md](权限模式切换-current.md) 与 [权限模式切换-api-current.md](权限模式切换-api-current.md)。字段级契约以 api 文档为准。
> 最后更新日期：2026-06-01

## 1. 核心业务规则

- `permissionMode` 会话级、内存态、不持久化；新会话默认 `default`。
- 运行中切换不打断当前轮，**下一轮 `query()` 生效**。
- 四枚举校验：default/acceptEdits/plan/bypassPermissions；非法回 error。
- `default` 走既有 `canUseTool` 弹框，行为零变化；其余三档交给 SDK 原生 `options.permissionMode`。

## 2. 接口入口指针（契约见 api 文档）

| 消息 | 实现类#方法 |
|---|---|
| WS `setMode {mode}` | `ClaudeChatService#setMode`（新增） |
| WS `open {…, mode?}` | `ClaudeChatService#openSession`（改，带 mode） |
| sidecar `setMode` | `SessionManager#setMode` → `Session.permissionMode`（新增/改） |

## 3. 涉及类清单（全路径 + 方法签名 + 职责）

### sidecar（TypeScript）

- `sidecar/claude-agent/src/sessionManager.ts` [改]
  - `class Session`：加字段 `permissionMode: string = 'default'`。
  - `Session.runTurn`：`query` 的 `options` 加 `permissionMode: this.permissionMode`。
  - `Session` 构造或 start 时可接受初始 mode。
  - `SessionManager.setMode(id: string, mode: string): void` → `this.sessions.get(id)?.permissionMode = mode`。
  - `SessionManager.start(id, cwd, model, mode?)`：透传初始 mode 到 Session。
- `sidecar/claude-agent/src/server.ts` [改]
  - `case 'setMode'`：`manager.setMode(sessionId, msg.mode as string)`。
  - `case 'start'`：解析 `msg.mode` 传入 `manager.start`。

### 后端（Java，tool-claude-chat）

- `…claudechat.api.dto.ClientMessage` [改]
  - 新增 `record SetMode(String mode) implements ClientMessage {}` + `@JsonSubTypes.Type(SetMode.class, name="setMode")`。
  - `Open` record 加 `String mode` 字段（jackson 反序列化缺省为 null）。
- `…claudechat.config.ClaudeChatWebSocketHandler` [改]
  - sealed switch 加 `case ClientMessage.SetMode sm -> service.setMode(ws, sm)`。
- `…claudechat.service.ClaudeChatService` [改]
  - `SessionCtx` 加 `volatile String mode = "default"`。
  - `setMode(WebSocketSession ws, ClientMessage.SetMode msg)`：定位 ctx → 校验枚举 → `ctx.mode = mode` → `sidecar.setMode(ctx.sessionId, mode)`；非法值 `sendError(ws, 0, "BAD_MODE", ...)`。
  - `openSession`：`sidecar.startSession(sessionId, cwd, open.model(), normalizeMode(open.mode()))`。
  - 私有 `isValidMode(String)` / `normalizeMode(String)`（null/空 → "default"）。
- `…claudechat.service.SidecarClient` [改]
  - `setMode(String sessionId, String mode)` → `send(Map.of("type","setMode","sessionId",sessionId,"mode",nz(mode)))`。
  - `startSession(sessionId, cwd, model, mode)`：start map 加 `"mode"`。

### 前端（features/claude-chat）

- `types.ts` [改]：`export type PermissionMode = 'default'|'acceptEdits'|'plan'|'bypassPermissions'`；`ClientMessage` 的 `open` 加 `mode?`，新增 `{ type:'setMode'; mode: PermissionMode }`。
- `hooks/useClaudeChatSocket.ts` [改]：`mode` state（默认 'default'）；`setMode(m)` → 乐观更新 + `sendRaw({type:'setMode',mode:m})`；接口暴露 `mode` / `setMode`；`open(cwd, model?, mode?)`。
- `components/ModeSwitch.tsx` [新增]：`{ mode, onChange, disabled }`，点击循环或下拉四档，显示中文标签；bypassPermissions 醒目色。
- `pages/ChatPage.tsx` [改]：输入区接入 `<ModeSwitch mode={chat.mode} onChange={chat.setMode} />`。

## 4. 数据结构

- 无 DB 变更。`permissionMode` 仅在 sidecar Session（TS 字段）、Java SessionCtx（volatile String）、前端 state 三处内存态流转。

## 5. 重要约束与边界

- Java 侧务必校验枚举，避免把非法字符串透到 SDK。
- `default` 分支不得改动既有 `canUseTool` 调用。
- 运行中切换只改字段，禁止中断/重启当前 query。
- 注释中文（kai-toolbox 约定）。

## 6. 测试 / 验证要点

- 切 acceptEdits → 下一轮 Edit/Write 不弹框、Bash 仍弹框。
- 切 bypassPermissions → 全部不弹框；切回 default → 恢复弹框。
- plan → 只出方案不改文件。
- 运行中切换当前轮不受影响。
- 回归：default 与现状一致；sidecar tsc / 前端 typecheck / 后端 compile 通过。
