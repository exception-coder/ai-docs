# VS Code Tunnel — 编码摘要

> 对应设计文档：[VS Code Tunnel-current.md](VS Code Tunnel-current.md)
> 最后更新：2026-05-21

---

## 1. 核心业务规则（编码必读）

| ID | 规则 | 编码约束 |
|----|------|---------|
| R1 | 全局单例，同时仅一个 `code tunnel` 进程 | `TunnelManager` 字段 `private Process currentProcess`；`start()` 在非 STOPPED 直接 return 当前 status |
| R2 | 进程长驻，与前端解耦 | 不在任何 HTTP 请求关闭回调里销毁进程 |
| R3 | stop 优雅退出 5s 超时 | `process.destroy()` → `process.waitFor(5, SECONDS)` → 超时 `destroyForcibly()` |
| R4 | 状态切换 `synchronized` | `start()` / `stop()` / `onLineHit()` / `onProcessExit()` 共用 `this` 锁 |
| R6 | 设备码正则 `code\s+([A-Z0-9]{4}-[A-Z0-9]{4})` | `Pattern.compile("code\\s+([A-Z0-9]{4}-[A-Z0-9]{4})")` |
| R7 | 隧道 URL 正则 `https?://vscode\.dev/tunnel/\S+` | `Pattern.compile("https?://vscode\\.dev/tunnel/\\S+")` |
| R8 | tunnelName 白名单 `^[a-zA-Z0-9][a-zA-Z0-9-]{0,31}$` | Controller 入参校验，非法 400 |
| R9 | SSE 连接立即下发当前快照 | `/events` handler 在 `register` 后立即 `emitter.send(currentStatus)` |
| R11 | `@PreDestroy` 杀子进程 | `TunnelManager#shutdown()` 加 `@PreDestroy` 调 `stop()` |
| R12 | 不持久化 | 无 `Repository`、无 schema.sql、无 `@Entity` |
| R13 | 错误日志保留 stderr 最后 1KB | 维护一个 1024 字节的 `ByteBuffer` 环形缓冲；ERROR 时写入 `lastError` |

---

## 2. 接口入口（指针）

字段级契约（请求体、响应体、错误码）在本工具中由各 record 类自身承载；接口列表如下：

| 方法 | 路径 | 实现 |
|------|------|------|
| GET | `/api/vscode-tunnel/status` | `TunnelController#status()` |
| POST | `/api/vscode-tunnel/start` | `TunnelController#start(StartRequest)` |
| POST | `/api/vscode-tunnel/stop` | `TunnelController#stop()` |
| GET | `/api/vscode-tunnel/events` | `TunnelController#events()`（SSE，content-type `text/event-stream`） |

---

## 3. 涉及类清单（全路径 + 方法签名）

### 3.1 com.exceptioncoder.toolbox.vscodetunnel.config.VsCodeTunnelToolDescriptor

实现 `ToolDescriptor`，无业务方法。注意 `route()` 是接口的**必实现**方法。

```java
@Component
public class VsCodeTunnelToolDescriptor implements ToolDescriptor {
    @Override public String id()          { return "vscode-tunnel"; }
    @Override public String name()        { return "VS Code Tunnel"; }
    @Override public String icon()        { return "globe"; }
    @Override public String route()       { return "/tools/vscode-tunnel"; }
    @Override public String group()       { return "系统工具"; }
    @Override public String description() { return "通过 code tunnel 把本地 VS Code 暴露给手机浏览器访问"; }
    @Override public int order()          { return 40; }
}
```

### 3.2 com.exceptioncoder.toolbox.vscodetunnel.config.VsCodeTunnelProperties

```java
@ConfigurationProperties(prefix = "toolbox.vscode-tunnel")
public record VsCodeTunnelProperties(
    boolean enabled,
    String codePath,            // 默认 "code"
    String tunnelName,          // 默认 ${HOSTNAME:kai-pc}
    boolean acceptLicense,      // 默认 true
    long stopGraceMs,           // 默认 5000
    int errorTailBytes          // 默认 1024
) {}
```

### 3.3 com.exceptioncoder.toolbox.vscodetunnel.domain.TunnelState

```java
public enum TunnelState { STOPPED, STARTING, AUTH_REQUIRED, RUNNING, STOPPING, ERROR }
```

### 3.4 com.exceptioncoder.toolbox.vscodetunnel.domain.TunnelStatus

```java
public record TunnelStatus(
    TunnelState state,
    @Nullable String tunnelUrl,
    @Nullable String deviceCode,
    @Nullable String deviceLoginUrl,  // 默认 "https://github.com/login/device"
    @Nullable String tunnelName,
    @Nullable Long pid,
    @Nullable Instant startedAt,
    @Nullable String lastError
) {
    public static TunnelStatus stopped() {
        return new TunnelStatus(TunnelState.STOPPED, null, null, null, null, null, null, null);
    }
}
```

### 3.5 com.exceptioncoder.toolbox.vscodetunnel.api.dto.StartRequest

```java
public record StartRequest(@Nullable String tunnelName) {}
```

### 3.6 com.exceptioncoder.toolbox.vscodetunnel.api.TunnelController

```java
@RestController
@RequestMapping("/api/vscode-tunnel")
public class TunnelController {
    private static final Pattern TUNNEL_NAME = Pattern.compile("^[a-zA-Z0-9][a-zA-Z0-9-]{0,31}$");
    private static final String SSE_KEY_PREFIX = "vscode-tunnel:";

    private final TunnelManager manager;
    private final SseEmitterRegistry sseRegistry;
    private final VsCodeTunnelProperties props;

    public TunnelStatus status();                                 // GET /status
    public TunnelStatus start(@RequestBody StartRequest req);     // POST /start
    public TunnelStatus stop();                                   // POST /stop
    public SseEmitter events();                                   // GET /events
}
```

- `start()`：先用 `TUNNEL_NAME` 正则校验（R8），不匹配抛 `ResponseStatusException(400)`；委托 `manager.start(name)`
- `events()`：生成 `SSE_KEY_PREFIX + UUID.randomUUID()` 作为 SSE key → `sseRegistry.create(key)` 拿 emitter → 挂 `onCompletion/onTimeout/onError` 回调，全部调 `manager.unsubscribe(key)` → 最后 `manager.subscribe(key)`；subscribe 内部立即 publish 当前快照（R9）
- `ensureEnabled()`：开关关闭时 `/start` 与 `/events` 返回 503

### 3.7 com.exceptioncoder.toolbox.vscodetunnel.service.TunnelManager

```java
@Component
public class TunnelManager {
    private static final String SSE_EVENT_NAME = "status";

    private final VsCodeTunnelProperties props;
    private final TunnelLauncher launcher;
    private final SseEmitterRegistry sseRegistry;

    // SSE 订阅 keys 集合。SseEmitterRegistry 是 key→emitter 一对一，
    // 多连接广播必须由 manager 自己维护订阅集合
    private final Set<String> subscribers = ConcurrentHashMap.newKeySet();

    private TunnelStatus current = TunnelStatus.stopped();
    private Process process;
    private Thread parserThread;
    private TunnelOutputParser.TailBuffer tail;

    public synchronized TunnelStatus status();
    public synchronized TunnelStatus start(String tunnelName);            // R1: 非 STOPPED/ERROR 直接 return current
    public synchronized TunnelStatus stop();                              // R3: destroy → waitFor 5s → destroyForcibly
    @PreDestroy public void shutdown();                                   // R11: 关停时 stop()

    // SSE 订阅管理（由 TunnelController 调用）
    public synchronized void subscribe(String key);                       // 加入集合 + 立即 publish 当前快照
    public void unsubscribe(String key);                                  // 移除集合（emitter 自身由 SseEmitterRegistry 回收）

    // 内部回调（解析线程调用）
    synchronized void onDeviceCode(String code);                          // STARTING → AUTH_REQUIRED
    synchronized void onTunnelUrl(String url);                            // (STARTING|AUTH_REQUIRED) → RUNNING
    synchronized void onProcessExit(int code, String tail);               // *→ STOPPED 或 ERROR
    private void transitionTo(TunnelStatus next);                         // 写 current + 遍历 subscribers publish
}
```

**关键实现点：**
- `start()` 启动 process 后，用 `Thread.ofVirtual().name("vscode-tunnel-parser-" + pid).start(...)` 跑 `TunnelOutputParser.parse(process.inputStream, this::onDeviceCode, this::onTunnelUrl, tailBuffer)`
- `start()` 同步注册 `process.onExit().thenAccept(p -> onProcessExit(p.exitValue(), tail.snapshot()))`
- `transitionTo()` 必须把 `current` 写完后再遍历 `subscribers` 调 `sseRegistry.publish(key, "status", next)`，避免 SSE 拿到旧快照
- `stop()` 中等待用 `process.waitFor(props.stopGraceMs(), MILLISECONDS)`，超时调 `destroyForcibly()`；同步落回 STOPPED 状态（不依赖异步 onExit）

### 3.8 com.exceptioncoder.toolbox.vscodetunnel.service.TunnelLauncher

```java
@Component
public class TunnelLauncher {
    private final VsCodeTunnelProperties props;

    public Process spawn(String tunnelName) throws TunnelStartException;
}
```

**实现：**
```java
List<String> cmd = new ArrayList<>(List.of(
    props.codePath(), "tunnel",
    "--name", tunnelName
));
if (props.acceptLicense()) cmd.add("--accept-server-license-terms");
ProcessBuilder pb = new ProcessBuilder(cmd).redirectErrorStream(true);  // R5: stderr 合流
try {
    return pb.start();
} catch (IOException e) {
    throw new TunnelStartException(
        "未找到 code 命令，请确认 VS Code 已安装且 CLI 已加入 PATH", e);
}
```

### 3.9 com.exceptioncoder.toolbox.vscodetunnel.service.TunnelOutputParser

```java
public class TunnelOutputParser {
    private static final Pattern DEVICE_CODE = Pattern.compile("code\\s+([A-Z0-9]{4}-[A-Z0-9]{4})");
    private static final Pattern TUNNEL_URL  = Pattern.compile("https?://vscode\\.dev/tunnel/\\S+");

    public static void parse(InputStream in,
                             Consumer<String> onDeviceCode,
                             Consumer<String> onTunnelUrl,
                             RingBuffer tailBuffer);
}
```

**实现：**
- 用 `BufferedReader(new InputStreamReader(in, UTF_8))` 按行读
- 每行：先写 `tailBuffer`（用于 ERROR 状态的 lastError）；再分别匹配两个正则；命中只触发一次（命中 URL 后不再触发 deviceCode）
- IO 异常视为进程结束，return

### 3.10 com.exceptioncoder.toolbox.vscodetunnel.service.TunnelStartException

```java
public class TunnelStartException extends RuntimeException {
    public TunnelStartException(String message, Throwable cause) { super(message, cause); }
}
```

由 `GlobalExceptionHandler`（toolbox-common 已有）兜底，返回 500 + 友好消息。

---

## 4. 数据结构

无数据库。无 schema.sql。

内存中维护：
- `TunnelStatus current` — 当前状态快照
- `RingBuffer tailBuffer` — 1KB 环形缓冲，记录最近 stdout 尾段

---

## 5. 重要约束与边界

| 约束 | 强制处 |
|------|-------|
| 单例：第二次 `start` 必须幂等 | `TunnelManager#start` 首行 `if (current.state() != STOPPED && current.state() != ERROR) return current` |
| tunnelName 白名单校验 | `TunnelController#validateTunnelName` 抛 `ResponseStatusException(400)` |
| stop 超时强杀 | `process.waitFor(stopGraceMs, MILLISECONDS)` 返回 false 即 `destroyForcibly()` |
| 解析线程异常隔离 | 用 try-catch 包裹 parse 循环，异常时仅日志，不影响 manager 状态机 |
| SSE emitter 上限 | 复用 `SseEmitterRegistry` 自身保护；无需在本工具内重复校验 |
| `@PreDestroy` 必须无异常完成 | 内部捕获所有异常并日志，避免阻塞 Spring 关停 |

### 5.1 前端关键约束

- `useTunnelStatus` 必须先 `GET /status` 拿初始快照，再 `new EventSource('/api/vscode-tunnel/events')` 订阅
- EventSource 自动重连默认 3s；不需要手动实现重连逻辑
- "启动 / 停止"按钮在 `STARTING / STOPPING` 状态期间 disabled
- `AUTH_REQUIRED` 视图必须提供"打开 GitHub 登录页"按钮（target=_blank 打开 `deviceLoginUrl`）
- `RUNNING` 视图必须包含：URL 文本（可选中复制）、复制按钮、二维码（≥ 200×200 px）

---

## 6. 验证要点

| 场景 | 验证 |
|------|------|
| 冷启动（首次） | start → AUTH_REQUIRED（看到设备码） → 用户登录 → RUNNING（拿到 URL） |
| 热启动（凭证缓存） | start → STARTING → RUNNING（不经 AUTH_REQUIRED） |
| 重复 start | 第二次返回当前 status，进程数仍为 1 |
| stop 优雅 | code 进程在 5s 内自行退出，状态 STOPPED |
| stop 强杀 | mock 一个不响应 SIGTERM 的进程；验证 5s 后 SIGKILL 生效 |
| code 不存在 | 改 properties `code-path: nonexistent`；验证 ERROR 状态 + lastError 文案 |
| Spring 关停 | 启动 tunnel 后 Ctrl+C 服务；验证 `code.exe` 进程已消失（任务管理器） |
| tunnelName 非法 | 入参 `"foo bar"` → 400 |
| SSE 重连 | 启动后断开 EventSource，3s 后自动重连，立即收到快照 |
| 手机扫码 | 二维码用手机相机扫，跳转 vscode.dev/tunnel/<name>，登录同账号可见本地 VS Code |
