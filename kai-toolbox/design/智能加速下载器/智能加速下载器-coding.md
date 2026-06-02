# 智能加速下载器 · 编码摘要

> 配套设计文档：[智能加速下载器-current.md](智能加速下载器-current.md)
> 接口契约：[智能加速下载器-api-current.md](智能加速下载器-api-current.md)

---

## 1. 核心业务规则

- **per-segment fallback**：单片主链路重试耗尽 → 切到 backup 链路再试一轮；任务级链路绑定已废弃
- **失败不连坐**：单片 FAILED **不**立即触发 `failTask`，等所有片到终态后由 `maybeFinalize` 聚合判定
- **失败不主动关 HttpClient**：`failTask` 只设 `shouldStop=true`，异步 `scheduleContextCleanup` 等所有 future 跑完再 close；**直接 close 会形成 IOException: closed 雪崩**
- **read-idle 守门狗强制中断**：worker 共享 `ScheduledExecutorService`，每秒检查 `lastReadAt`，超过 `readIdleTimeoutMs`（默认 30s）就 `close(stream)` 让阻塞 `read()` 抛 IOException → 触发重试
- **重试 jitter 必须有**：退避公式 `min(base*2^attempt, maxMs) + random(-50%, +50%)`，多片同时失败时错开重试时刻
- **自适应并发只降不升**：失败率 > 50% 或 HTTP 429 → 永久 acquire 部分 permits 把 `effectiveParallel` 砍半；不主动加并发回去（避免抖动）；10s 冷却
- **HTTP/2 优先**：`HttpClient.newBuilder().version(HTTP_2)`；ALPN 协商失败 JDK 自动回退 H1
- **race 不复用 body**：256KB 测速字节**不落盘**，避免 CDN 返回 200（忽略 Range 头）时把整文件流前段误写入临时文件；探测成功后正式下载从 offset=0 重新拉
- 没有 `Accept-Ranges: bytes` 或无 `Content-Range` 时降级为单分片顺序下载
- 默认保存目录：`<user.home>/Downloads/kai-toolbox/`；目标文件已存在追加 `(1)/(2)` 后缀
- Windows 文件名非法字符 `< > : " | ? * \ /` 一律替换为 `_`
- 进程启动时所有 `PROBING` / `DOWNLOADING` 任务**强制转为 PAUSED**，不自动续传
- 任务状态切换必须经 `DownloaderTaskService#transitionTo(taskId, newState)` 单一入口
- 状态机：`QUEUED ⇄ {PROBING, PAUSED, FAILED}`；`DOWNLOADING ⇄ {PAUSED, COMPLETED, FAILED}`；`PAUSED → {DOWNLOADING, FAILED}`；**`FAILED → QUEUED` 允许（resume 触发重试）**
- SSE `progress` 事件以 500ms 时间窗聚合可丢；`state` 事件必送达
- 单片重试 6 次（1s/2s/4s/8s/16s/30s，封顶 30s + ±50% jitter），单任务总耗时上限 ~60s
- 单任务默认 4 路并发，全局上限 16；自适应触发后只降不升

---

## 2. 接口入口指针

> 字段级契约见 `智能加速下载器-api-current.md`，本节仅列实现定位。

| 接口 | 实现类 #方法 |
|------|-------------|
| `POST /api/downloader/tasks` | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#createTask` |
| `GET /api/downloader/tasks` | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#listTasks` |
| `GET /api/downloader/tasks/{id}` | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#getTask` |
| `POST /api/downloader/tasks/{id}/pause` | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#pauseTask` |
| `POST /api/downloader/tasks/{id}/resume` | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#resumeTask` |
| `DELETE /api/downloader/tasks/{id}` | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#deleteTask` |
| `GET /api/downloader/tasks/{id}/events` (SSE) | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#subscribeEvents` |
| `GET /api/downloader/proxy/detect` | `com.exceptioncoder.toolbox.downloader.api.DownloaderController#detectProxy` |

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `kai-toolbox/pom.xml` | 修改 | 注册 `<module>tools/tool-downloader</module>` + dependencyManagement 条目 |
| `kai-toolbox/tools/tool-downloader/pom.xml` | 新建 | 模块 pom，依赖 toolbox-common + spring-boot-starter-web |
| `com.exceptioncoder.toolbox.downloader.api.DownloaderController` | 新建 | REST + SSE 入口（薄壳） |
| `com.exceptioncoder.toolbox.downloader.api.dto.CreateTaskRequest` | 新建 | record(url, savePath, filename) |
| `com.exceptioncoder.toolbox.downloader.api.dto.TaskView` | 新建 | 任务列表/详情视图 record |
| `com.exceptioncoder.toolbox.downloader.api.dto.SegmentView` | 新建 | 分片视图 record |
| `com.exceptioncoder.toolbox.downloader.api.dto.RouteDecisionView` | 新建 | race 结果视图 record |
| `com.exceptioncoder.toolbox.downloader.api.dto.ProxyProbeResult` | 新建 | /proxy/detect 返回体 record |
| `com.exceptioncoder.toolbox.downloader.api.dto.ProxyCandidateView` | 新建 | 代理候选 record |
| `com.exceptioncoder.toolbox.downloader.config.DownloaderToolDescriptor` | 新建 | 工具菜单注册 |
| `com.exceptioncoder.toolbox.downloader.config.DownloaderProperties` | 新建 | `@ConfigurationProperties("toolbox.downloader")` |
| `com.exceptioncoder.toolbox.downloader.config.DownloaderConfig` | 新建 | 注册 `HttpClient` Bean、虚拟线程 Executor、应用启动钩子 |
| `com.exceptioncoder.toolbox.downloader.domain.DownloadTask` | 新建 | 任务实体 |
| `com.exceptioncoder.toolbox.downloader.domain.DownloadSegment` | 新建 | 分片实体 |
| `com.exceptioncoder.toolbox.downloader.domain.TaskState` | 新建 | 枚举 |
| `com.exceptioncoder.toolbox.downloader.domain.SegmentState` | 新建 | 枚举 |
| `com.exceptioncoder.toolbox.downloader.domain.RouteType` | 新建 | 枚举 DIRECT / PROXY |
| `com.exceptioncoder.toolbox.downloader.domain.ProxyCandidate` | 新建 | 代理候选 record(source, host, port, originUrl) |
| `com.exceptioncoder.toolbox.downloader.domain.RouteDecision` | 新建 | 链路决策 record |
| `com.exceptioncoder.toolbox.downloader.domain.TaskRepository` | 新建 | SQLite DAO（JdbcTemplate） |
| `tool-downloader/src/main/resources/db/downloader-schema.sql` | 新建 | 建表 SQL，由 toolbox-common 的 `SchemaInitializer` 自动加载 |
| `com.exceptioncoder.toolbox.downloader.service.TaskService` | 新建 | 编排 + 状态机 |
| `com.exceptioncoder.toolbox.downloader.service.ProxyDetector` | 新建 | 系统代理探测 |
| `com.exceptioncoder.toolbox.downloader.service.RouteProber` | 新建 | 直连/代理 race |
| `com.exceptioncoder.toolbox.downloader.service.SegmentDownloader` | 新建 | 分片 worker |
| `com.exceptioncoder.toolbox.downloader.service.ProgressBus` | 新建 | 进度窗口聚合（500ms），通过 `SseEmitterRegistry.publish` 推送 |
| `com.exceptioncoder.toolbox.downloader.service.FilenameResolver` | 新建 | URL/响应头 → 安全文件名 |
| `toolbox-starter/src/main/resources/application.yml` | 修改 | 追加 `toolbox.downloader.*` 配置节点（不新建 application-downloader.yml） |
| `toolbox-starter/pom.xml` | 修改 | 引入 tool-downloader 依赖 |
| `kai-toolbox/frontend/src/pages/DownloaderPage.tsx` | 新建 | 整页 |
| `kai-toolbox/frontend/src/components/downloader/NewTaskForm.tsx` | 新建 | URL 表单，主 CTA lg+shadow |
| `kai-toolbox/frontend/src/components/downloader/TaskList.tsx` | 新建 | 列表 |
| `kai-toolbox/frontend/src/components/downloader/TaskCard.tsx` | 新建 | 单卡片 |
| `kai-toolbox/frontend/src/components/downloader/ProxyStatusBadge.tsx` | 新建 | 顶部代理状态 |
| `kai-toolbox/frontend/src/hooks/useDownloaderSSE.ts` | 新建 | EventSource 封装 |
| `kai-toolbox/frontend/src/api/downloader.ts` | 新建 | REST 客户端 |

### 关键方法签名与职责

```text
# Controller
DownloaderController#createTask(@Valid CreateTaskRequest req): ResponseEntity<ApiResult<TaskView>>
    — 校验 url，委托 TaskService.create，返回 201 + TaskView
DownloaderController#listTasks(@RequestParam(required=false) String state, @RequestParam(defaultValue="50") int limit): ApiResult<List<TaskView>>
DownloaderController#getTask(@PathVariable long id): ApiResult<TaskDetailView>
DownloaderController#pauseTask(@PathVariable long id): ApiResult<TaskView>
DownloaderController#resumeTask(@PathVariable long id): ApiResult<TaskView>
DownloaderController#deleteTask(@PathVariable long id, @RequestParam(defaultValue="false") boolean keepFile): ResponseEntity<Void>
DownloaderController#subscribeEvents(@PathVariable long id): SseEmitter
    — 注册到 ProgressBus，超时设 0（无限）
DownloaderController#detectProxy(): ApiResult<ProxyProbeResult>

# DownloaderTaskService（@Service） — 编排 + 状态机 + 自适应并发
DownloaderTaskService#create(String url, String savePath, String filename): DownloadTask
    — 入参校验 → 写入 DB(QUEUED) → 异步 safeKickoff
DownloaderTaskService#listAll(Set<TaskState> filter, int limit): List<DownloadTask>
DownloaderTaskService#findById(long id): DownloadTask
DownloaderTaskService#pause(long id): DownloadTask     — 标记 PAUSED + 设 shouldStop=true
DownloaderTaskService#resume(long id): DownloadTask    — PAUSED → 续传未完成片；FAILED → 重置所有非 DONE 片为 PENDING，状态 → QUEUED 重新走 kickoff
DownloaderTaskService#delete(long id, boolean keepFile): void   — 关闭主备 HttpClient，删 .kdownload
DownloaderTaskService#transitionTo(long id, TaskState next, String error): void    — 状态机单一入口
DownloaderTaskService#restoreOnStartup(): void          — @PostConstruct，扫描 PROBING/DOWNLOADING → PAUSED

# 私有：编排
private DownloaderTaskService#kickoff(long taskId, String userFilename): void
    — race 探测 → 写 routeDecision/totalSize/acceptRanges/filename → 切片 → 构造 RuntimeContext(primary+backup) → dispatch
private DownloaderTaskService#resumeWorkers(long taskId): void
    — 已有 route 时直接重建 RuntimeContext 拉起 worker（备用 client 重新探测系统代理，因为期间用户可能开/关 VPN）
private DownloaderTaskService#dispatchSegments(...): void
    — 给每个 PENDING/DOWNLOADING 片提交一个 worker 到 workerExecutor (虚拟线程)
private DownloaderTaskService#runSegment(...): void
    — acquire parallelGate → Phase 1: primaryClient × retryMax/2 次
                          → Phase 2: backupClient × retryMax/2 次（若 primary 失败且有 backup）
                          → 落库 + 累加自适应窗口 + 触发 maybeAdaptParallelism
                          → DONE 走 onSegmentDone / FAILED 走 onSegmentFailed
                          → finally release parallelGate

# 私有：失败聚合与自适应
private DownloaderTaskService#onSegmentDone(long taskId, Path finalPath): void   — 调 maybeFinalize
private DownloaderTaskService#onSegmentFailed(long taskId, String err): void     — 调 maybeFinalize（不立即失败任务）
private DownloaderTaskService#maybeFinalize(long taskId): void
    — 全 DONE → finalizeTask；全到终态且有 FAILED → failTask(聚合错误「N/M 片失败：...」)；否则等
private DownloaderTaskService#maybeAdaptParallelism(long taskId, RuntimeContext ctx): void
    — 窗口完成 ≥4 片或失败 ≥2 片时检查；失败率 > 50% → shrinkParallel；10s 冷却
private DownloaderTaskService#shrinkParallel(RuntimeContext ctx, int target, String reason): void
    — 永久 tryAcquire(reduce) 个 permit 不再 release，等价降低 Semaphore 上限
private DownloaderTaskService#failTask(long taskId, String error): void
    — transitionTo FAILED；设 shouldStop=true；**不**主动 close HttpClient；调 scheduleContextCleanup
private DownloaderTaskService#scheduleContextCleanup(long taskId): void
    — 异步等所有 worker future 完成（最长 10s/片）再 close HttpClient，避免雪崩
private DownloaderTaskService#cleanupContext(long taskId): void
    — close primaryClient + backupClient，从 runtimeContexts 移除

# ProxyDetector（@Service）
ProxyDetector#detect(): List<ProxyCandidate>
    — 合并三源：JVM property → env(HTTPS_PROXY/HTTP_PROXY) → ProxySelector.getDefault().select(URI)
ProxyDetector#effective(URI target): Optional<ProxyCandidate>
    — 同 detect 但只返回第一优先级（用于实际下载）

# RouteProber（@Service）
RouteProber#race(URI url, Optional<ProxyCandidate> proxy): RaceResult
    — 同时发 2 个 GET Range 0-262143，先完成 256KB 的胜
    — 返回 RaceResult(RouteDecision decision, HttpHeaders winnerHeaders)；headers 用于解析 Content-Length / Accept-Ranges / Content-Disposition
RouteProber.ProbeOutcome（内部 record）：success, ttfbMs, throughputBps, headers, error

# HttpClientFactory（@Component）
HttpClientFactory#newClient(Optional<ProxyCandidate>): HttpClient    — 走代理的 client
HttpClientFactory#newDirectClient(): HttpClient                      — NO_PROXY 直连 client
private HttpClientFactory#baseBuilder(): HttpClient.Builder
    — 统一构造：HTTP/2 优先 + connectTimeout + followRedirects.NORMAL + 虚拟线程 executor

# SegmentDownloader（@Service） — 含 read-idle 守门狗
final ScheduledExecutorService watchdog   — 单线程 daemon scheduler，所有 worker 共享
SegmentDownloader#download(HttpClient client, URI url, DownloadSegment seg, FileChannel fc,
                           LongConsumer onBytes, BooleanSupplier shouldStop): SegmentOutcome
    — 使用默认 retryMax (props.segmentRetryMax)
SegmentDownloader#download(HttpClient client, URI url, DownloadSegment seg, FileChannel fc,
                           LongConsumer onBytes, BooleanSupplier shouldStop, int maxRetries): SegmentOutcome
    — 自定义 retryMax 重载，per-segment fallback 时调用方传 retryMax/2
    — 内部循环：失败时 backoff = min(base*2^attempt, maxMs) + ±50% jitter
private SegmentDownloader#downloadOnce(...): void
    — 发 GET Range from-to；429 抛 HttpStatus429Exception；非 206/200 抛 IOException
    — try-with 持有 AtomicLong lastReadAt
    — 提交 scheduleAtFixedRate watchdog：每秒检查 now - lastReadAt > readIdleTimeoutMs → close(in)
    — read 循环：每次 read 更新 lastReadAt；检查 shouldStop；写入 fc.write(buf, writeOffset)；progressCallback 累加
    — finally cancel watchdog

# ProgressBus（@Service） — 复用 toolbox-common 的 SseEmitterRegistry，不重造 emitter
ProgressBus#addBytes(long taskId, long delta): void                     — AtomicLong 累加到窗口
ProgressBus#publishState(long taskId, TaskState s, RouteDecision r, String err): void
                                                                         — 调 sseRegistry.publish(key, "state", payload)
ProgressBus#publishSegment(long taskId, DownloadSegment seg): void      — sseRegistry.publish(key, "segment", payload)
ProgressBus#closeTask(long taskId): void                                — sseRegistry.complete(key)
@Scheduled(fixedRate=500) ProgressBus#flush(): void                     — 遍历活跃任务，构造 progress payload 调 sseRegistry.publish
SSE key 约定：String.valueOf(taskId)

# FilenameResolver
FilenameResolver#resolve(URI url, HttpHeaders headers): String   — 优先 Content-Disposition; 否则 url 末段；sanitize
FilenameResolver#deduplicate(Path dir, String name): Path        — 同名追加 (1) (2)

# TaskRepository（@Repository）
insertTask(DownloadTask) / updateTaskState(long, TaskState, String error) / updateRouteDecision(long, RouteDecision)
listAll(Set<TaskState>, int) / findById(long) / deleteTask(long)
insertSegments(long taskId, List<DownloadSegment>)
listSegmentsByTask(long taskId)
updateSegment(DownloadSegment)
sumDownloadedBytes(long taskId): long

# DownloaderToolDescriptor（@Component）
id "downloader" / name "智能加速下载器" / icon "download" / route "/tools/downloader" / group "网络工具" / order 25
```

---

## 4. 数据结构

### 表 `tool_downloader_task`

> 文件落点：`tool-downloader/src/main/resources/db/downloader-schema.sql`，由 toolbox-common 的 `SchemaInitializer` 启动时自动加载。**必须 `IF NOT EXISTS`，幂等。**

```sql
CREATE TABLE IF NOT EXISTS tool_downloader_task (
    id                    INTEGER PRIMARY KEY AUTOINCREMENT,
    url                   TEXT    NOT NULL,
    save_path             TEXT    NOT NULL,
    filename              TEXT    NOT NULL,
    total_size            INTEGER NOT NULL DEFAULT -1,         -- -1 表示未知
    accept_ranges         INTEGER NOT NULL DEFAULT 0,          -- 0/1
    state                 TEXT    NOT NULL,                    -- QUEUED/PROBING/DOWNLOADING/PAUSED/COMPLETED/FAILED
    route_type            TEXT,                                -- DIRECT/PROXY/NULL
    route_proxy           TEXT,                                -- http://127.0.0.1:7890
    probe_direct_ttfb_ms  INTEGER,
    probe_direct_bps      INTEGER,
    probe_proxy_ttfb_ms   INTEGER,
    probe_proxy_bps       INTEGER,
    last_error            TEXT,
    created_at            TEXT    NOT NULL,                    -- ISO-8601
    updated_at            TEXT    NOT NULL
);
```

### 表 `tool_downloader_segment`

```sql
CREATE TABLE IF NOT EXISTS tool_downloader_segment (
    task_id           INTEGER NOT NULL,
    seq_no            INTEGER NOT NULL,
    offset_bytes      INTEGER NOT NULL,
    length_bytes      INTEGER NOT NULL,
    bytes_downloaded  INTEGER NOT NULL DEFAULT 0,
    state             TEXT    NOT NULL,                        -- PENDING/DOWNLOADING/DONE/FAILED
    attempts          INTEGER NOT NULL DEFAULT 0,
    last_error        TEXT,
    PRIMARY KEY (task_id, seq_no),
    FOREIGN KEY (task_id) REFERENCES tool_downloader_task(id) ON DELETE CASCADE
);
CREATE INDEX IF NOT EXISTS idx_segment_task ON tool_downloader_segment(task_id);
```

### 关键 DTO/Record 字段

```java
public record CreateTaskRequest(
    @NotBlank String url,
    String savePath,                 // 可空，走默认值
    String filename                  // 可空，自动解析
) {}

public record TaskView(
    long taskId, String url, String savePath, String filename,
    long totalSize, long downloadedSize, String state,
    String routeType, String routeProxy,
    long currentRateBps, Long etaSeconds,
    String createdAt, String updatedAt
) {}

public record ProxyCandidate(Source source, String host, int port, String originUrl) {
    public enum Source { JVM_PROPERTY, ENV, WINDOWS_REGISTRY }
}

public record RouteDecision(
    RouteType route,
    Long directTtfbMs, Long directThroughputBps,
    Long proxyTtfbMs,  Long proxyThroughputBps,
    String decidedAt
) {}
```

### 配置默认值（合并到 `toolbox-starter/src/main/resources/application.yml` 的 `toolbox.downloader` 节点）

```yaml
toolbox:
  downloader:
    default-save-path: ${user.home}/Downloads/kai-toolbox
    segment-size: 33554432               # 32 MiB
    max-parallel-per-task: 4             # 4 是国内 CDN 稳定性甜点；8+ 容易触发限流
    max-parallel-global: 16
    probe-timeout-ms: 8000
    probe-bytes: 262144                  # 256 KiB
    segment-retry-max: 6                 # 6 次扛过 CDN 偶发 20-60 秒小风暴
    segment-retry-backoff-base-ms: 1000  # 退避序列 1s/2s/4s/8s/16s/30s 加 ±50% jitter
    segment-retry-backoff-max-ms: 30000  # 退避封顶 30s
    sse-flush-interval-ms: 500
    connect-timeout-ms: 10000
    request-timeout-ms: 0                # 0 = 不限制单次请求总耗时
    read-idle-timeout-ms: 30000          # ★ 治 stalled bug 的核心配置，详见 current.md 第 10.1 节
    proxy: ${TOOLBOX_HTTP_PROXY:}        # 留空走环境变量探测
```

---

## 5. 重要约束与边界

- **状态切换单入口**：任何位置修改 `tool_downloader_task.state` 必须经 `DownloaderTaskService#transitionTo`
- **行级锁**：`ConcurrentHashMap<Long, Object>` 自封装；按 taskId 串行化状态机操作
- **预分配文件**：进入 DOWNLOADING 前 `RandomAccessFile.setLength(total)`；不支持 Range 且 totalSize=-1 时跳过
- **写入并发**：每个 worker 用独立 `FileChannel.open(WRITE)` + `position(offset)` 写入对应区间；不共享 channel
- **HttpClient 双 client**：`RuntimeContext` 持 `primaryClient` + `backupClient`，单片粒度 per-segment fallback；任务结束时**异步**等所有 worker 完成后再 close（防止雪崩，见 current.md §10.2）
- **协作式中断**：每次 read 后检查 `shouldStop`；read 阻塞时由守门狗 `close(stream)` 强制中断（见 current.md §10.1）
- **read-idle 守门狗**：所有 worker 共享一个 daemon `ScheduledExecutorService`，每秒检查；每片只占 1 个 `ScheduledFuture`，开销可忽略
- **重试退避**：6 次，`min(base*2^attempt, 30s) + ±50% jitter`；attempts 持久化到 segment 表
- **per-segment fallback**：单片 primary 失败 → 切 backup 重试。两段各 `retryMax/2` 次；无 backup 时合并为 `retryMax` 次都用 primary
- **失败聚合**：单片 FAILED 不立即 failTask；所有片到终态后由 `maybeFinalize` 聚合判定，错误信息形如「3/11 片失败：xxx」
- **自适应并发**：`Semaphore` permits = effectiveParallel；降并发 = 永久 acquire 不还。窗口完成 ≥4 片或失败 ≥2 片才检查；10s 冷却防误判
- **限流降级**：检测 429 → `shrinkParallel(current/2)` + 2s 后重排该片；最低降到 1
- **进程启动钩子**：`@PostConstruct restoreOnStartup` 把 PROBING/DOWNLOADING 强制转 PAUSED，不自动续传
- **SSE emitter 清理**：emitter 关闭/异常时由 toolbox-common 的 `SseEmitterRegistry` 主动移除；任务终态时 `closeTask` → `sse.complete`
- **HTTP/2 优先**：`HttpClient.Version.HTTP_2`，协商失败 JDK 自动回退 H1
- **不处理的场景**：HTTPS 证书不受信（不 trust-all）、PAC 脚本、需要 Cookie 的 URL、`ftp://` 等非 HTTP scheme、TLS 指纹混淆

---

## 6. 下游依赖调用

```text
// JDK 内置（无第三方）
java.net.http.HttpClient
java.net.ProxySelector / ProxySelector.getDefault()
java.net.http.HttpRequest.newBuilder().method("GET", BodyPublishers.noBody())
java.nio.channels.FileChannel
java.util.concurrent.Executors.newVirtualThreadPerTaskExecutor()

// toolbox-common 复用
com.exceptioncoder.toolbox.common.tool.ToolDescriptor       — ToolDescriptor 接口
com.exceptioncoder.toolbox.common.api.ApiResult             — 若存在；否则用 Map<String,Object> 包装
com.exceptioncoder.toolbox.common.persistence.*             — SQLite/JdbcTemplate（参考 tool-projects / video-library 现状）
```

---

## 7. 异常处理要点

- 参数校验失败（`@Valid`）→ `MethodArgumentNotValidException` → toolbox-common GlobalExceptionHandler 返回 400
- URL 非 http(s) scheme / savePath 非法 → `IllegalArgumentException` → 400
- 两条链路全部不可达 → `RouteProber.UnreachableException`（`@ResponseStatus(BAD_GATEWAY)`）→ 502，**任务不写入 DB**
- 任务 ID 不存在 → `DownloaderTaskService.TaskNotFoundException`（`@ResponseStatus(NOT_FOUND)`）→ 404
- resume 不可恢复（COMPLETED）→ `TaskNotResumableException`（`@ResponseStatus(CONFLICT)`）→ 409
  - **已知坑**：toolbox-common GlobalExceptionHandler 的 `handleAny(Exception)` 兜底优先级高于 `@ResponseStatus` 注解，会把它当成 500 处理。**当前规避方式**：`resume(FAILED)` 直接走重试路径不抛异常，所以实际不会触发；如未来其他工具遇到同样问题，需修 GlobalExceptionHandler 让它尊重 `@ResponseStatus`
- 分片下载 `IOException` → 进入退避重试；耗尽返回 `SegmentOutcome.failed`
- 分片下载触发 **守门狗 close** → `read()` 抛 `IOException("Stream closed")` → 走重试（这是我们对 stalled 的兜底，详见 current.md §10.1）
- 分片下载 **`PausedException`**（内部信号）→ `SegmentOutcome.paused` (state=PENDING)，**不**触发 onSegmentFailed
- 429 → `SegmentDownloader.HttpStatus429Exception` 上抛 → `runSegment` catch → `shrinkParallel(current/2)` + 2s 后重排
- `Content-Range` 校验失败 → 探测阶段不复用 body，正式下载阶段视为 IOException 触发重试
- `FileChannel` write 失败（磁盘满等）→ 该片 FAILED，由 `maybeFinalize` 决定是否拖垮任务
- SSE emitter `send` 抛 IOException（客户端断开）→ `SseEmitterRegistry` 主动移除 emitter，不影响下载
- **failTask 不立即 close HttpClient**：防止「一片失败让所有兄弟片 IOException: closed」雪崩（详见 current.md §10.2）

---

## 8. 实现顺序建议

1. 父 pom 注册 module + 模块 pom
2. domain 枚举与 record（无依赖）
3. SQLite migration SQL + TaskRepository（可单测）
4. ProxyDetector + FilenameResolver（纯函数式，可单测）
5. RouteProber + 配套 HttpClient 工厂（用 mock URL 单测）
6. SegmentDownloader（独立 worker 单测）
7. ProgressBus（SSE 测试需 emitter）
8. TaskService（编排层，集成 1-7）
9. DownloaderController + DTO
10. DownloaderToolDescriptor + DownloaderProperties + DownloaderConfig
11. toolbox-starter pom 接入
12. 前端 api 客户端 + hooks + 4 个组件 + 1 个页面
13. 端到端测试：下载小文件、暂停恢复、进程重启、限流场景

---

## 9. 故障速查表

| 现象 | 最可能根因 | 排查点 |
|------|----------|--------|
| 速率 0 但状态还是 DOWNLOADING，永远不变 | **stalled bug**：worker 卡在 `read()`，守门狗没生效 | 查 `SegmentDownloader.downloadOnce` 的 watchdog 块是否真的提交了；`readIdleTimeoutMs` 是否被外部配置改成 0；现象详见 current.md §10.1 |
| 1 片真失败 + N 片 IOException: closed 雪崩 | `failTask` 又开始主动 close HttpClient 了 | 检查 `failTask` 是否还在直接调 `cleanupContext`，应该走 `scheduleContextCleanup` 异步路径；详见 current.md §10.2 |
| 1 片失败立即任务 FAILED，其他片白跑 | `onSegmentFailed` 直接调了 `failTask` 而不是 `maybeFinalize` | 检查 `onSegmentFailed` 是否经 `maybeFinalize` 聚合；详见 current.md §10.3 |
| 重试 3 次都失败但服务端其实活着 | 同步重试撞同一波拒绝窗口，缺 jitter | 检查 `SegmentDownloader.download` 的 backoff 是否加了 `(Math.random() - 0.5)` jitter；详见 current.md §10.4 |
| 4 路并发被服务端限流，但 Chrome 单连接没事 | HTTP/1.1 多 socket 被指纹识别 | 检查 `HttpClientFactory.baseBuilder` 是否带 `.version(HTTP_2)`；详见 current.md §10.5 |
| 服务端就是单 IP 限并发，怎么试都 50% 失败 | 应该触发自适应降并发但没生效 | 检查 `windowTotal` / `windowFailed` 累加点；检查 `shrinkParallel` 是否真的 `tryAcquire(reduce)` 成功；详见 current.md §10.6 |
| Lark URL 一直 8/11 片失败，VPN 这条没用上 | per-segment fallback 没工作 | 检查 `RuntimeContext.backupClient` 是否非 null；检查 `runSegment` Phase 2 切换分支；详见 current.md §10.7 |
| resume FAILED 任务返回 500 而不是 409 | toolbox-common `GlobalExceptionHandler.handleAny` 优先级高于 `@ResponseStatus` | 当前 resume(FAILED) 已绕过此问题（不抛异常直接重试）；如出现需修 GlobalExceptionHandler |
| `tool_downloader_task.last_error` 永远是英文异常类名 | 错误信息没本地化 | 在 `failTask` / `maybeFinalize` 聚合错误时已经做了「N/M 片失败：...」前缀；如要更精细需翻译 IOException message |
| 进程重启后任务自动开始下载 | `restoreOnStartup` 没工作或被绕过 | 检查 `@PostConstruct` 是否生效；`DownloaderTaskService` 是否被 Spring 真的初始化（看启动日志） |
