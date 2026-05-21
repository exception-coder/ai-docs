# Web 终端 - 接口契约（WebSocket）

> 最后更新：2026-05-09
> 本文档承载本工具对外的唯一接口契约：1 个 WebSocket Endpoint。

---

## 1. 端点

| 项 | 值 |
|---|---|
| 路径 | `/api/webterm/ws` |
| 协议 | WebSocket（文本帧，UTF-8 JSON） |
| 鉴权 | 无（与项目整体一致） |
| 入站消息上限 | 64 KB / 帧 |
| 出站消息上限 | 8 KB（缓冲触发推送） |
| 同进程并发上限 | 8 个会话 |

---

## 2. 消息格式

所有消息均为单层 JSON，必含 `type` 字段以区分类型。前端→后端称 **ClientMessage**，后端→前端称 **ServerMessage**。

### 2.1 ClientMessage

#### 2.1.1 `open`（建立会话，连接后必发，仅一次）

```json
{
  "type": "open",
  "shell": "powershell",
  "cwd": "C:\\Users\\zhang",
  "cols": 120,
  "rows": 30
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| type | string | 是 | 固定 `"open"` |
| shell | string | 是 | 白名单：`"powershell"` / `"cmd"` |
| cwd | string \| null | 否 | 工作目录绝对路径；为空则用 `${user.home}` |
| cols | int | 是 | 终端列数（≥20、≤500） |
| rows | int | 是 | 终端行数（≥5、≤200） |

#### 2.1.2 `input`（用户按键流）

```json
{
  "type": "input",
  "data": "dir\r\n"
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| data | string | 是 | 原样（含控制字符）写入子进程 stdin；UTF-8 |

#### 2.1.3 `resize`（终端尺寸变化）

```json
{
  "type": "resize",
  "cols": 160,
  "rows": 42
}
```

#### 2.1.4 `close`（前端主动结束会话）

```json
{ "type": "close" }
```

收到后服务端会强杀进程并回复 `exit` 后关闭 WebSocket。

---

### 2.2 ServerMessage

#### 2.2.1 `ready`（进程已启动，可以输入了）

```json
{
  "type": "ready",
  "sessionId": "8e3d-ab02-...",
  "shell": "powershell",
  "cwd": "C:\\Users\\zhang",
  "pid": 12345
}
```

#### 2.2.2 `output`（stdout/stderr 输出，不区分流）

```json
{
  "type": "output",
  "data": "PS C:\\Users\\zhang> "
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| data | string | UTF-8 字符串；可能被分片，前端按到达顺序 `term.write` 即可 |

#### 2.2.3 `exit`（进程已退出）

```json
{
  "type": "exit",
  "code": 0
}
```

收到后服务端会立即关闭 WebSocket（关闭码 1000）。

#### 2.2.4 `error`（协议或启动错误）

```json
{
  "type": "error",
  "code": "SHELL_LAUNCH_FAILED",
  "message": "powershell.exe not found on PATH"
}
```

| `code` | 含义 |
|---|---|
| `OPEN_REQUIRED` | 入站第一条消息不是 `open` |
| `OPEN_DUPLICATED` | 同一连接收到第二个 `open` |
| `SESSION_LIMIT_EXCEEDED` | 已达 `max-sessions` 上限 |
| `UNSUPPORTED_PLATFORM` | 当前 OS 不在白名单（非 Windows） |
| `INVALID_SHELL` | 非白名单 shell |
| `INVALID_CWD` | cwd 不是已存在目录 |
| `MESSAGE_TOO_LARGE` | 入站消息超 64KB |
| `SHELL_LAUNCH_FAILED` | `ProcessBuilder.start()` 抛异常 |
| `INTERNAL_ERROR` | 兜底；详情写后端日志 |

收到 `error` 后服务端可能立即关闭 WebSocket（关闭码 1011），前端需识别此码做出"会话已结束"的展示。

---

## 3. 时序约束

```text
Client                              Server
  | -- WS Connect ------------------> |
  |                                   |  握手成功
  | <- WS Established ----------------|
  | -- {type:"open", ...} ----------> |
  |                                   |  ProcessBuilder.start
  | <- {type:"ready", ...} ---------- |
  | <- {type:"output", ...} --------- |  欢迎横幅 / PS1
  |                                   |
  | -- {type:"input", ...} ---------> |  反复
  | <- {type:"output", ...} --------- |  反复
  | -- {type:"resize", ...} --------> |
  |                                   |
  | -- {type:"close"} --------------> |  或浏览器关闭
  | <- {type:"exit", code:N} -------- |
  | <- WS Close (1000) -------------- |
```

- **首条消息必须是 `open`**：连接后 5 秒内未收到 `open`，服务端关连接（关闭码 1008）
- **`output` 完全无序保证**：服务端按读到的顺序串行推送；前端无需自己排序
- **`exit` 之后**：服务端不再发送任何消息；前端收到后视为终态

---

## 4. 关闭码语义

| 关闭码 | 触发原因 |
|---|---|
| 1000 | 正常退出（`exit` 后） |
| 1008 | 协议违反（首条非 `open` / 重复 `open` / 入站超大） |
| 1011 | 服务端启动 Shell 失败或内部错误（与 `error` 配对） |
| 1001 | 浏览器导航离开 / 关闭页面 |

---

## 5. 配置项

`application.yml`：

| 键 | 默认 | 说明 |
|---|---|---|
| `toolbox.webterm.enabled` | `true` | 关闭后 Endpoint 直接拒绝握手 |
| `toolbox.webterm.default-shell` | `powershell` | 前端未传 shell 时兜底 |
| `toolbox.webterm.max-sessions` | `8` | 同进程并发会话上限 |
| `toolbox.webterm.output-buffer-bytes` | `8192` | 单次推送字节阈值 |
| `toolbox.webterm.output-flush-interval-ms` | `50` | 推送时间阈值 |
| `toolbox.webterm.session-idle-timeout-ms` | `0` | 0 = 不超时 |
