# 权限模式切换 — 接口契约

> 配套 [权限模式切换-current.md](权限模式切换-current.md)。本期只扩展既有 WS 协议 `/api/claude-chat/ws`，无新增 HTTP 端点。
> 最后更新日期：2026-06-01

---

## 1. 权限模式枚举

```ts
type PermissionMode = 'default' | 'acceptEdits' | 'plan' | 'bypassPermissions'
```

| 值 | 中文标签 | 语义 |
|----|---------|------|
| `default` | 默认 | 每个工具弹权限框等决策（现状） |
| `acceptEdits` | 自动接受 | 编辑类工具自动放行，其余仍弹框 |
| `plan` | 计划 | 只读规划，不执行改动 |
| `bypassPermissions` | 全自动 | 全部自动放行，不弹框 |

非法值由 Java 侧拒绝并回 `error {code: BAD_MODE}`。

---

## 2. WS 协议扩展（客户端 → 服务端）

### 2.1 新增 `setMode`

运行中或任意时刻切换当前会话权限模式。

```ts
| { type: 'setMode'; mode: PermissionMode }
```

```java
// ClientMessage.java
record SetMode(String mode) implements ClientMessage {}
```

- 服务端处理：`ClaudeChatService.setMode` 校验枚举 → 更新 `SessionCtx.mode` → `SidecarClient.setMode(sessionId, mode)`。
- **下一轮生效**：当前正在跑的一轮不受影响。

### 2.2 `open` 增加可选 `mode`

新建会话可带初始模式，不传按 `default`。

```ts
| { type: 'open'; cwd: string; model?: string; mode?: PermissionMode }
```

```java
record Open(String cwd, String model, String mode) implements ClientMessage {}
```

---

## 3. Java ↔ sidecar 协议

| 方向 | 消息 |
|------|------|
| Java → sidecar | `{type:'setMode', sessionId, mode}` |
| Java → sidecar | `{type:'start', sessionId, cwd, model, mode}`（`start` 增加 `mode` 字段） |

sidecar：`server.ts` 路由到 `SessionManager.setMode(id, mode)` → `Session.permissionMode = mode`；`runTurn` 时 `query({ options: { permissionMode } })`。

> 服务端 → 客户端事件（`ServerMessage`）本期**不新增**；前端 `mode` 由本地乐观更新维护。如需服务端权威回显，后续可加 `modeChanged` 事件（见 current §8）。
