# Web 终端 - 编码摘要

> 与 `Web终端-current.md` 配套；仅承载方法级实现要点。
> 接口字段级契约见 `Web终端-api-current.md`。

---

## 1. 核心业务规则（实现约束）

- 一 WebSocket = 一 Session = 一 Process，绑定不可拆分；任意一方关闭立即销毁另两方
- 同进程并发会话上限 `toolbox.webterm.max-sessions`（默认 8），命中即回 `error: SESSION_LIMIT_EXCEEDED`
- 入站首条必须是 `open`，5 秒未到关连接（1008）；后续 `open` 视为协议违反
- 入站消息体超 64KB 关连接（1008）；DTO 反序列失败回 `INTERNAL_ERROR`
- 输出按"8KB 缓冲 / 50ms 时间"二选一触发推送（`output-buffer-bytes` / `output-flush-interval-ms`）
- 仅支持 Windows 平台启动 PowerShell/cmd；非 Windows 直接回 `UNSUPPORTED_PLATFORM`
- PowerShell 启动时强制 UTF-8 输出编码；cmd 启动时 `chcp 65001`
- 移动端辅助键、快捷命令、历史命令都只能通过 `TerminalHandle.send` 写入 PTY，不能本地 echo 到 xterm
- 移动端底部操作区必须保留 44px 级触控目标，并使用 `100dvh` / safe-area 降低软键盘遮挡

---

## 2. 接口入口

| 端点 | 实现类#方法 | 备注 |
|------|------|------|
| `WS /api/webterm/ws` | `WebTermSocketHandler#afterConnectionEstablished/handleTextMessage/afterConnectionClosed` | 字段级契约见 api-doc |

---

## 3. 涉及类清单

### 3.1 `com.exceptioncoder.toolbox.webterm.config.WebTermDescriptor`

实现 `ToolDescriptor`：
- `id() → "webterm"`，`name() → "Web 终端"`，`icon() → "terminal-square"`
- `route() → "/tools/webterm"`，`group() → "系统工具"`
- `description() → "在浏览器中打开 PowerShell / cmd 命令行"`，`order() → 30`

### 3.2 `com.exceptioncoder.toolbox.webterm.config.WebTermProperties`

`@ConfigurationProperties("toolbox.webterm")`，字段：
- `enabled: boolean = true`
- `defaultShell: String = "powershell"`
- `maxSessions: int = 8`
- `outputBufferBytes: int = 8192`
- `outputFlushIntervalMs: long = 50`
- `sessionIdleTimeoutMs: long = 0`

### 3.3 `com.exceptioncoder.toolbox.webterm.config.WebTermWebSocketConfig`

`@Configuration @EnableWebSocket implements WebSocketConfigurer`：
- `registerWebSocketHandlers(WebSocketHandlerRegistry registry)` → 注册 `webTermSocketHandler` 到 `/api/webterm/ws`，`setAllowedOrigins("*")`
- `@Bean public ServletServerContainerFactoryBean` → `setMaxTextMessageBufferSize(64 * 1024)`、`setMaxBinaryMessageBufferSize(64 * 1024)`

### 3.4 `com.exceptioncoder.toolbox.webterm.handler.WebTermSocketHandler`

继承 `TextWebSocketHandler`：

```java
public void afterConnectionEstablished(WebSocketSession ws)
// 设置 ws.setTextMessageSizeLimit(64*1024)
// 调度 5 秒超时任务：若仍未 open 则 close(1008, "OPEN_REQUIRED")

protected void handleTextMessage(WebSocketSession ws, TextMessage msg)
// JSON 反序列化为 ClientMessage
// switch on type:
//   open  -> handleOpen(ws, openMsg)
//   input -> session.writeStdin(data)
//   resize -> session.setSize(cols, rows)
//   close -> closeSession(ws, ServerMessage.exit(code=-1))

private void handleOpen(WebSocketSession ws, OpenMessage msg)
// 1. 校验 OS = Windows，否则 sendError(UNSUPPORTED_PLATFORM) + close(1011)
// 2. 校验 shell ∈ {powershell, cmd}，否则 INVALID_SHELL
// 3. 校验 cwd（null 用 user.home；非空必须 Files.isDirectory）
// 4. registry.tryAcquireSlot()，满则 SESSION_LIMIT_EXCEEDED
// 5. ShellLauncher.launch(shell, cwd) → Process
// 6. new WebTermSession(ws, process, props)，registry.register
// 7. session.startOutputForwarding(this::sendOutput)
// 8. sendReady(sessionId, shell, cwd, pid)

public void afterConnectionClosed(WebSocketSession ws, CloseStatus s)
// registry.findByWs(ws)?.close()
// registry.remove(ws)
```

### 3.5 `com.exceptioncoder.toolbox.webterm.session.WebTermSession`

```java
final String sessionId        // UUID
final WebSocketSession ws
final PtyProcess process      // pty4j 提供，IS-A java.lang.Process
final WebTermProperties props
final AtomicBoolean closed
volatile int cols, rows
final Object stdinLock = new Object()
final Object wsSendLock = new Object()
Thread stdoutThread           // 仅 1 条；stderr 已合流
```

方法：
- `startOutputForwarding()`：用 `Thread.ofVirtual().start(...)` 起 1 条线程读 PtyProcess.inputStream，循环 `read(byte[N])`；累积到 `output-buffer-bytes` 或 `output-flush-interval-ms` 触发转 UTF-8 调用 `sendMessage`；EOF 退出
- `writeStdin(String data)`：`synchronized(stdinLock)` 内 `process.outputStream.write(bytes)` + flush；进程已死则丢弃
- `setSize(int c, int r)`：记录字段后调 `process.setWinSize(new WinSize(c, r))` 真下发到 PTY，让 PowerShell 重排行宽
- `sendMessage(ServerMessage)`：`synchronized(wsSendLock)` 内 `ws.sendMessage(new TextMessage(json))`；ws 已关时跳过
- `waitForExit()`：单独虚拟线程 `process.waitFor()`；退出后等 stdoutThread join 500ms 让残留 buffer flush，再发 `exit` 消息并 `close()`
- `close()`：CAS 防重入；`process.destroyForcibly()`；中断 stdoutThread；关 WS

### 3.6 `com.exceptioncoder.toolbox.webterm.session.WebTermSessionRegistry`

```java
final Map<String, WebTermSession> byId = new ConcurrentHashMap<>()
final Map<WebSocketSession, WebTermSession> byWs = new ConcurrentHashMap<>()
final WebTermProperties props

boolean tryAcquireSlot()             // size < maxSessions
void register(WebTermSession s)
WebTermSession findByWs(WebSocketSession ws)
void remove(WebSocketSession ws)     // 同时删 byId/byWs

@PreDestroy
void closeAll()                      // forEach session.close()
```

### 3.7 `com.exceptioncoder.toolbox.webterm.service.ShellLauncher`

```java
PtyProcess launch(String shell, Path cwd, int cols, int rows) throws IOException
void resize(PtyProcess process, int cols, int rows)
```

实现：
- 校验 `System.getProperty("os.name")` 包含 `windows`
- 命令数组（不再用 `-NoExit -Command` 设置编码——PTY 模式下 PowerShell 自动按 console 编码工作）：
  - `powershell.exe -NoLogo -NoProfile -ExecutionPolicy Bypass`
  - `cmd.exe /K chcp 65001>nul`
- `PtyProcessBuilder()`
  - `.setCommand(command)`
  - `.setDirectory(cwd.toString())`
  - `.setEnvironment(env)` —— `env.putIfAbsent("TERM", "xterm-256color")`
  - `.setInitialColumns(cols)` / `.setInitialRows(rows)`
  - `.setConsole(false)` —— 启用 ConPTY 伪终端
  - `.setUseWinConPty(true)` —— Windows 显式用 ConPTY，旧 winpty 不再用
  - `.setRedirectErrorStream(true)` —— stderr 合流到 stdout，与本地终端一致
  - `.start()`
- `resize(process, cols, rows)` 调 `process.setWinSize(new WinSize(cols, rows))`

### 3.8 `com.exceptioncoder.toolbox.webterm.api.dto.ClientMessage` / `ServerMessage`

`sealed interface` + `permits` 子记录，配合 Jackson `@JsonTypeInfo(use = NAME, property = "type")` + `@JsonSubTypes`：

```java
sealed interface ClientMessage permits Open, Input, Resize, Close {
  record Open(String shell, String cwd, int cols, int rows) implements ClientMessage {}
  record Input(String data) implements ClientMessage {}
  record Resize(int cols, int rows) implements ClientMessage {}
  record Close() implements ClientMessage {}
}

sealed interface ServerMessage permits Ready, Output, Exit, Error {
  record Ready(String sessionId, String shell, String cwd, long pid) implements ServerMessage {}
  record Output(String data) implements ServerMessage {}
  record Exit(int code) implements ServerMessage {}
  record Error(String code, String message) implements ServerMessage {}
}
```

---

## 4. 数据结构

无数据库变更。所有状态保留在 JVM 进程内（`WebTermSessionRegistry`）。

---

## 5. 重要约束与边界

- **线程模型**：handler 线程仅做 IO 转发；输出读取放虚拟线程；Tomcat WebSocket 自身要求同一 Session 写不可并发，所以 `wsSendLock` 必须串行
- **资源回收**：Process 销毁后必须中断 stdout/stderr 读线程（`Thread.interrupt()` + 阻塞 `read` 通过 `process.destroyForcibly()` 关闭流自然解除）
- **编码**：所有 `byte[] → String` 显式 `new String(bytes, 0, len, StandardCharsets.UTF_8)`
- **空值**：`cwd` 反序列化为 `null` 时使用 `System.getProperty("user.home")`
- **取消超时任务**：`open` 收到后必须取消 5s 超时 future，否则会误关连接
- **错误消息发送顺序**：`error` → `close(1011)`，前端能收到完整 error；不要先 close 再 sendMessage（会失败）

---

## 6. 前端实现要点

### 6.1 `frontend/src/features/webterm/index.tsx`

```tsx
import { TerminalSquare } from 'lucide-react'
const manifest: FeatureManifest = {
  id: 'webterm',
  name: 'Web 终端',
  icon: TerminalSquare,
  group: '系统工具',
  description: '在浏览器中打开 PowerShell / cmd 命令行',
  order: 30,
  routes: [{ path: '/tools/webterm', element: <WebTermPage /> }],
}
export default manifest
```

### 6.2 `frontend/src/features/webterm/hooks/useWebTermSocket.ts`

接口：
```ts
type State = 'idle' | 'connecting' | 'opening' | 'ready' | 'closed' | 'error'
function useWebTermSocket(opts: {
  enabled: boolean
  shell: 'powershell' | 'cmd'
  cwd?: string
  cols: number
  rows: number
  onOutput: (data: string) => void
  onExit: (code: number) => void
  onError: (code: string, message: string) => void
}): { state: State; send: (data: string) => void; resize: (c, r) => void; close: () => void; reconnectKey: number }
```

实现：
- `useEffect` 内 `new WebSocket(\`${proto}//${host}/api/webterm/ws\`)`，`proto = location.protocol === 'https:' ? 'wss:' : 'ws:'`
- onopen：发送 `open`；切换到 `opening`
- onmessage：JSON 解析；switch type 调对应回调；`ready` 切到 `ready`
- onclose / onerror：切到 `closed` / `error`
- cleanup：`ws.close()`

### 6.3 `frontend/src/features/webterm/components/Terminal.tsx`

- `useEffect` mount 时：
  ```ts
  const term = new Terminal({ cursorBlink: true, fontFamily: 'Consolas, Menlo, monospace', fontSize: 13, theme: {...} })
  const fit = new FitAddon(); term.loadAddon(fit)
  term.open(divRef.current); fit.fit()
  term.onData(data => { term.write(data); /* 本地 echo for RK2 */ socket.send(data) })
  ```
- `ResizeObserver` → `fit.fit()` → `socket.resize(term.cols, term.rows)`
- 收到 output → `term.write(data)`
- 收到 exit/error → `term.writeln('\r\n[会话已结束]')`
- unmount：`term.dispose()`

### 6.4 `frontend/src/features/webterm/pages/WebTermPage.tsx`

- 顶部一行控件：Shell 单选（PowerShell / cmd）、cwd 输入框、`重新连接` 按钮、`结束进程` 按钮
- 主体：`<Terminal shell={shell} cwd={cwd} key={reconnectKey} />`
- 不支持平台时（捕获到 `UNSUPPORTED_PLATFORM`），显示提示卡片代替终端
- 移动端页面根使用 `100dvh`，终端主体保持 `flex-1 min-h-0`，底部只放辅助键与命令工作台

### 6.5 `frontend/src/features/webterm/components/AuxKeyBar.tsx`

- 按 Shell 高频编辑路径排序：Esc、Tab、Enter、方向键、Ctrl+A/E/U/K、Backspace、Ctrl+C、Ctrl+L、Home、End
- 每个按钮 `h-11 min-w-11`，横向滚动承载窄屏溢出
- `onPointerDown` 对按钮阻止默认行为，避免点辅助键时 xterm 隐藏输入框失焦、软键盘收起

### 6.6 `frontend/src/features/webterm/components/MobileCommandInput.tsx`

- `quickCommands` 渲染为横向滚动 chip，点按后直接发送预置命令字节
- 文本输入使用 textarea；Enter 发送当前命令，Shift+Enter 保留换行；发送时补 `\r`
- 成功发送后写入 `localStorage["webterm_cmd_history"]`，最多保留 50 条并去重
- 历史面板选择命令只回填输入框，不立即执行，避免误触执行危险命令
- 剪贴板读取失败不改变输入内容；读取成功后追加到当前输入并重新聚焦

---

## 7. 风险与验证

- **手工验证场景**：
  1. 打开页面 → 看到 PowerShell PS1 横幅
  2. 输入 `dir` → 看到当前目录列表
  3. 输入中文路径 `dir D:\\项目\\` → 中文不乱码
  4. 切换到 cmd → 重连 → 输入 `chcp` 应回 65001
  5. 关闭浏览器标签 → 任务管理器中对应 powershell.exe 应消失
  6. 连续打开 9 个页面 → 第 9 个应收到 `SESSION_LIMIT_EXCEEDED`
  7. 输入 `exit` → 收到 `exit` 消息且 WS 关闭
- **不验证（已声明限制）**：vim、htop、git log 分页、Read-Host 后端回显
