# 编码摘要文档：LLM 任务管理

> 配套设计文档：`llm-task-management-current.md`
> 接口契约：`llm-task-management-api-current.md`
> 本文档只回答"每个方法怎么写"，不重复设计决策与流程图。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-05-25 | 初始版本，落库 + 事件双写 + 重启恢复 + 断线重连 |
| v2 | 2026-05-25 | 增补：6 个 SPI 默认实现 + 自动装配 + @LlmTaskExecutorType 注解，对应主设计 v2 章节 10 |

---

## 1. 核心业务规则

- R1 write-through：状态变更必须先写 DB 成功，再更新 cache；DB 失败不污染 cache
- R2 状态机单向：`PENDING→QUEUED→RUNNING→{COMPLETED|FAILED|CANCELLED|TIMED_OUT}`、`RUNNING→INTERRUPTED`（仅恢复路径）；违法迁移抛 `IllegalTaskStateException`
- R3 event_seq 串行分配：`UPDATE llm_task SET last_event_seq=last_event_seq+1 ... RETURNING`；`(task_id, event_seq)` 唯一索引兜底
- R4 启动恢复：扫 `status IN (QUEUED, RUNNING)`，写 INTERRUPTED 事件，按 max_retries 决定重调度
- R5 协作式取消：写 `cancellation_requested=1`，executor 主动 `checkCancellation()`；禁止 `Thread.interrupt()`
- R6 缓存淘汰：活跃永驻，终态保留 `task.retain-minutes`（默认 30）后淘汰
- R7 事件双写：过程事件批量异步落盘（50 条 / 1 秒触发），**终态事件强制同步落盘**
- R8 token 不入库：token 流走独立 `Flux<String>` 通道
- R9 业务侧禁直连 Repository：必须通过 `LlmTaskService`
- R10 幂等：`idempotency_key` 唯一约束，重复提交返回已存在任务

---

## 2. 接口入口指针

字段级契约见 `llm-task-management-api-current.md`。

| 接口 | 实现类#方法 |
|------|-------------|
| `GET /api/v1/tasks` | `com.exceptioncoder.llm.api.controller.management.LlmTaskController#list` |
| `GET /api/v1/tasks/{taskId}` | `LlmTaskController#getTask` |
| `GET /api/v1/tasks/{taskId}/events` | `LlmTaskController#listEvents` |
| `GET /api/v1/tasks/{taskId}/stream` | `LlmTaskController#stream` |
| `POST /api/v1/tasks/{taskId}/cancel` | `LlmTaskController#cancel` |

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.llm.domain.task.model.LlmTask` | 新建 | record + Builder + 状态机校验静态方法 |
| `com.exceptioncoder.llm.domain.task.model.LlmTaskStatus` | 新建 | 枚举 8 个状态 |
| `com.exceptioncoder.llm.domain.task.model.LlmTaskEvent` | 新建 | record |
| `com.exceptioncoder.llm.domain.task.model.LlmTaskEventType` | 新建 | 枚举 10 个事件类型 |
| `com.exceptioncoder.llm.domain.task.model.LlmTaskSubmitCommand` | 新建 | 提交命令 |
| `com.exceptioncoder.llm.domain.task.model.LlmTaskContext` | 新建 | 执行上下文 |
| `com.exceptioncoder.llm.domain.task.repository.LlmTaskRepository` | 新建 | 接口 |
| `com.exceptioncoder.llm.domain.task.repository.LlmTaskEventRepository` | 新建 | 接口 |
| `com.exceptioncoder.llm.domain.task.service.LlmTaskEventPublisher` | 新建 | 事件总线接口 |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskExecutor` | 新建 | SPI 1 - 业务执行（必选） |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskExecutorRegistry` | 新建 | task_type → executor 路由 |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskExecutorType` | 新建 | @Target(TYPE) @Retention(RUNTIME) 注解，value=task_type |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskInterceptor` | 新建 | SPI 2 - 4 个钩子方法（default 实现为空） |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskIdGenerator` | 新建 | SPI 3 - generate(taskType) → String |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskCancellationStrategy` | 新建 | SPI 4 - canCancel / shouldCheck checkpoint 判定 |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskRetryPolicy` | 新建 | SPI 5 - shouldRetry / nextDelayMs |
| `com.exceptioncoder.llm.domain.task.spi.LlmTaskAuditLogger` | 新建 | SPI 6 - 4 个审计事件方法 |
| `com.exceptioncoder.llm.domain.task.exception.IllegalTaskStateException` | 新建 | |
| `com.exceptioncoder.llm.domain.task.exception.TaskNotFoundException` | 新建 | |
| `com.exceptioncoder.llm.domain.task.exception.TaskCancelledException` | 新建 | |
| `com.exceptioncoder.llm.domain.task.exception.TaskTimeoutException` | 新建 | |
| `com.exceptioncoder.llm.application.task.LlmTaskService` | 新建 | 外观：submit/get/cancel/streamEvents |
| `com.exceptioncoder.llm.application.task.LlmTaskQueryService` | 新建 | 分页查询 |
| `com.exceptioncoder.llm.infrastructure.task.repository.JpaLlmTaskRepository` | 新建 | JPA 实现 + 乐观锁 |
| `com.exceptioncoder.llm.infrastructure.task.repository.JpaLlmTaskEventRepository` | 新建 | JPA 实现 + 批量 insert |
| `com.exceptioncoder.llm.infrastructure.task.repository.entity.LlmTaskEntity` | 新建 | JPA 实体 |
| `com.exceptioncoder.llm.infrastructure.task.repository.entity.LlmTaskEventEntity` | 新建 | JPA 实体 |
| `com.exceptioncoder.llm.infrastructure.task.repository.jpa.LlmTaskJpaSpringRepository` | 新建 | Spring Data |
| `com.exceptioncoder.llm.infrastructure.task.repository.jpa.LlmTaskEventJpaSpringRepository` | 新建 | Spring Data |
| `com.exceptioncoder.llm.infrastructure.task.cache.LlmTaskCache` | 新建 | Caffeine LRU |
| `com.exceptioncoder.llm.infrastructure.task.publisher.ReactorLlmTaskEventPublisher` | 新建 | Sinks 热流 + 异步落盘 |
| `com.exceptioncoder.llm.infrastructure.task.publisher.EventFlushWorker` | 新建 | @Scheduled flush |
| `com.exceptioncoder.llm.infrastructure.task.pool.LlmTaskExecutorPool` | 新建 | 按 task_type 分组线程池 |
| `com.exceptioncoder.llm.infrastructure.task.pool.LlmTaskTimeoutWatcher` | 新建 | 超时守护 |
| `com.exceptioncoder.llm.infrastructure.task.pool.LlmTaskAsyncConfig` | 新建 | 配置 + Bean 装配 |
| `com.exceptioncoder.llm.infrastructure.task.restore.LlmTaskRestoreRunner` | 新建 | ApplicationRunner |
| `com.exceptioncoder.llm.infrastructure.task.restore.LlmTaskCleanupScheduler` | 新建 | @Scheduled 清理 |
| `com.exceptioncoder.llm.infrastructure.task.spi.DefaultTaskIdGenerator` | 新建 | SPI 3 默认实现 |
| `com.exceptioncoder.llm.infrastructure.task.spi.CooperativeCancellationStrategy` | 新建 | SPI 4 默认实现 |
| `com.exceptioncoder.llm.infrastructure.task.spi.ExponentialBackoffRetryPolicy` | 新建 | SPI 5 默认实现 |
| `com.exceptioncoder.llm.infrastructure.task.spi.Slf4jAuditLogger` | 新建 | SPI 6 默认实现 |
| `com.exceptioncoder.llm.infrastructure.task.spi.LlmTaskInterceptorChain` | 新建 | 多 Interceptor 链式包装 |
| `com.exceptioncoder.llm.infrastructure.task.autoconfigure.LlmTaskAutoConfiguration` | 新建 | @AutoConfiguration 主入口 |
| `com.exceptioncoder.llm.infrastructure.task.autoconfigure.LlmTaskWebAutoConfiguration` | 新建 | Web 端点装配（独立条件 ConditionalOnClass(SseEmitter)） |
| `com.exceptioncoder.llm.infrastructure.task.autoconfigure.LlmTaskProperties` | 新建 | @ConfigurationProperties("llm.task") |
| `com.exceptioncoder.llm.infrastructure.task.autoconfigure.LlmTaskCoreBeansRegistrar` | 新建 | 注册 Service/Pool/Publisher/Cache 默认 Bean，全部 @ConditionalOnMissingBean |
| `com.exceptioncoder.llm.infrastructure.task.autoconfigure.LlmTaskInterceptorRegistrar` | 新建 | 收集 Spring 容器内所有 LlmTaskInterceptor 形成链 |
| `com.exceptioncoder.llm.infrastructure.task.autoconfigure.LlmTaskExecutorRegistrar` | 新建 | BeanPostProcessor 扫描 @LlmTaskExecutorType 注解建路由表 |
| `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` | 新建 | 列 2 个 AutoConfiguration 类，启用自动装配 |
| `com.exceptioncoder.llm.api.controller.management.LlmTaskController` | 新建 | REST 入口 |
| `com.exceptioncoder.llm.api.controller.management.dto.LlmTaskStatusResponse` | 新建 | |
| `com.exceptioncoder.llm.api.controller.management.dto.LlmTaskEventResponse` | 新建 | |
| `com.exceptioncoder.llm.api.controller.management.dto.LlmTaskListResponse` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.agent.executor.AgentExecutor` | 修改 | 实现 `LlmTaskExecutor`，标注 `@LlmTaskExecutorType("AGENT")` |
| `com.exceptioncoder.llm.infrastructure.agent.task.AgentTaskManagerImpl` | 修改 | 改为薄壳委托给 `LlmTaskService`（过渡期双轨） |
| `com.exceptioncoder.llm.infrastructure.devplan.graph.DevPlanGraphExecutor` | 修改 | 实现 `LlmTaskExecutor`，标注 `@LlmTaskExecutorType("DEVPLAN")` |
| `com.exceptioncoder.llm.infrastructure.devplan.control.DevPlanTaskManagerImpl` | 修改 | 改为薄壳委托 |
| `db/migration/V20260525__llm_task.sql` | 新建 | Flyway 建表脚本 |

### 关键方法签名与职责

```
// === Domain 层 SPI（v2 新增） ===
com.exceptioncoder.llm.domain.task.spi.LlmTaskExecutor#execute(LlmTask task, LlmTaskContext ctx): String
  — 业务方实现，返回 finalOutputJson；执行中需频繁 ctx.checkCancellation() / ctx.publish() / ctx.updateProgress()

com.exceptioncoder.llm.domain.task.spi.LlmTaskExecutorType (annotation)
  — @Target(TYPE) @Retention(RUNTIME) String value()
  — 标在 @Component 上，由 LlmTaskExecutorRegistrar 在启动时扫描

com.exceptioncoder.llm.domain.task.spi.LlmTaskInterceptor (default methods)
  beforeSubmit(LlmTaskSubmitCommand cmd): LlmTaskSubmitCommand  // 默认返回原 cmd，可改写
  beforeExecute(LlmTask task): void                              // 默认空实现
  afterComplete(LlmTask task, String result): void              // 默认空实现
  onError(LlmTask task, Throwable error): void                  // 默认空实现

com.exceptioncoder.llm.domain.task.spi.LlmTaskIdGenerator#generate(String taskType): String
com.exceptioncoder.llm.domain.task.spi.LlmTaskCancellationStrategy#canCancel(LlmTask task): boolean
com.exceptioncoder.llm.domain.task.spi.LlmTaskCancellationStrategy#shouldCheckpoint(LlmTask task): boolean
com.exceptioncoder.llm.domain.task.spi.LlmTaskRetryPolicy#shouldRetry(LlmTask task, Throwable err): boolean
com.exceptioncoder.llm.domain.task.spi.LlmTaskRetryPolicy#nextDelayMs(int retryCount): long
com.exceptioncoder.llm.domain.task.spi.LlmTaskAuditLogger#onSubmit/onComplete/onCancel/onFail(LlmTask): void

// === Infrastructure 层 SPI 默认实现（v2 新增） ===
DefaultTaskIdGenerator#generate(taskType): String
  — Map.of("AGENT","ag","DEVPLAN","dp","RESUME","rs",...).get(taskType) + "-" + yyyyMMdd + "-" + AtomicLong

CooperativeCancellationStrategy#canCancel(task): boolean
  — 仅活跃状态可取消（PENDING/QUEUED/RUNNING）

ExponentialBackoffRetryPolicy#shouldRetry(task, err): boolean
  — task.retryCount < task.maxRetries && !(err instanceof TaskCancelledException)

ExponentialBackoffRetryPolicy#nextDelayMs(retryCount): long
  — Math.min(base * (1L << retryCount), maxDelayMs)，默认 base=1000 / max=60000

Slf4jAuditLogger#onSubmit(task): void
  — LoggerFactory.getLogger("llm.task.audit").info("SUBMIT taskId={} type={} user={} ", ...)

LlmTaskInterceptorChain implements LlmTaskInterceptor
  — 构造：List<LlmTaskInterceptor> ordered by @Order
  — 4 个钩子顺序遍历调用；beforeSubmit 链式改写 cmd

// === Spring Boot 自动装配（v2 新增） ===
LlmTaskAutoConfiguration
  @AutoConfiguration
  @ConditionalOnProperty(prefix="llm.task", name="enabled", havingValue="true", matchIfMissing=true)
  @EnableConfigurationProperties(LlmTaskProperties.class)
  @Import({CoreBeansRegistrar, InterceptorRegistrar, ExecutorRegistrar})

LlmTaskCoreBeansRegistrar 内每个 Bean：
  @Bean @ConditionalOnMissingBean
  LlmTaskService llmTaskService(repo, eventRepo, publisher, pool, idGen, retry, audit, interceptorChain) { ... }
  (同理 LlmTaskRepository / LlmTaskEventRepository / LlmTaskCache / LlmTaskEventPublisher / LlmTaskExecutorPool / LlmTaskIdGenerator / LlmTaskCancellationStrategy / LlmTaskRetryPolicy / LlmTaskAuditLogger 各自一个默认 Bean)

LlmTaskExecutorRegistrar implements BeanPostProcessor
  postProcessAfterInitialization: 检查 bean 是否有 @LlmTaskExecutorType + implements LlmTaskExecutor
    → registry.register(annotation.value(), bean)
    → 重复 type 抛 DuplicateExecutorTypeException

// === Domain 层（v1） ===
com.exceptioncoder.llm.domain.task.model.LlmTask#withStatus(LlmTaskStatus next): LlmTask
  — 不可变迁移，校验当前→next 合法；非法抛 IllegalTaskStateException

com.exceptioncoder.llm.domain.task.model.LlmTask#withProgress(int percent, String step): LlmTask
  — 进度更新，percent 限 0~100

com.exceptioncoder.llm.domain.task.model.LlmTask#incrementRetry(): LlmTask
  — retry_count + 1

com.exceptioncoder.llm.domain.task.model.LlmTaskStatus.transitions: Map<LlmTaskStatus, Set<LlmTaskStatus>>
  — 静态合法迁移表

com.exceptioncoder.llm.domain.task.model.LlmTaskContext#publish(LlmTaskEventType type, Object payload): void
  — 业务方调用，转发到 LlmTaskEventPublisher

com.exceptioncoder.llm.domain.task.model.LlmTaskContext#checkCancellation(): void
  — 查 task.cancellation_requested，true 抛 TaskCancelledException

com.exceptioncoder.llm.domain.task.model.LlmTaskContext#updateProgress(int percent, String step): void
  — 更新 task.progress + 发布 PROGRESS 事件

com.exceptioncoder.llm.domain.task.repository.LlmTaskRepository#save(LlmTask task): LlmTask
  — INSERT 或 UPDATE，写完同步 cache

com.exceptioncoder.llm.domain.task.repository.LlmTaskRepository#updateStatus(String taskId, LlmTaskStatus expectedCurrent, LlmTaskStatus next): boolean
  — 乐观锁 UPDATE ... WHERE task_id=? AND status=?，返回是否更新成功；false 时表示状态已被其他线程改

com.exceptioncoder.llm.domain.task.repository.LlmTaskRepository#requestCancel(String taskId): boolean
  — UPDATE ... SET cancellation_requested=1 WHERE status IN ('PENDING','QUEUED','RUNNING')

com.exceptioncoder.llm.domain.task.repository.LlmTaskRepository#findInterrupted(int limit): List<LlmTask>
  — SELECT ... WHERE status IN ('QUEUED','RUNNING') LIMIT ?

com.exceptioncoder.llm.domain.task.repository.LlmTaskEventRepository#appendBatch(List<LlmTaskEvent> events): void
  — JdbcTemplate batch insert

com.exceptioncoder.llm.domain.task.repository.LlmTaskEventRepository#findByTaskIdAfterSeq(String taskId, long afterSeq, int limit): List<LlmTaskEvent>
  — SELECT ... WHERE task_id=? AND event_seq>? ORDER BY event_seq ASC LIMIT ?

com.exceptioncoder.llm.domain.task.service.LlmTaskEventPublisher#publish(String taskId, LlmTaskEventType type, String dataJson): LlmTaskEvent
  — 1) 串行分配 seq（DB UPDATE...RETURNING 或应用层锁）2) 投递 Sinks 热流 3) 终态事件同步落盘 / 过程事件入异步队列；返回事件含分配的 seq

com.exceptioncoder.llm.domain.task.service.LlmTaskEventPublisher#subscribe(String taskId): Flux<LlmTaskEvent>
  — 返回订阅时点之后的热流；任务终态时流自动 complete

com.exceptioncoder.llm.domain.task.service.LlmTaskExecutor#execute(LlmTask task, LlmTaskContext ctx): String
  — 业务方实现，返回 finalOutput（JSON）；执行中应频繁 ctx.checkCancellation() 与 ctx.publish()

com.exceptioncoder.llm.domain.task.service.LlmTaskExecutorRegistry#resolve(String taskType): LlmTaskExecutor
  — 按 @LlmTaskExecutorType 注解扫描注册，未匹配抛 IllegalStateException

// === Application 层 ===
com.exceptioncoder.llm.application.task.LlmTaskService#submit(LlmTaskSubmitCommand cmd): LlmTask
  — 1) 幂等检查 findByIdempotencyKey 2) tryAcquireSlot(taskType) 3) save(PENDING) 4) pool.submit 5) 返回 PENDING 任务
  — 槽位满抛 ConcurrencyExceededException（API 转 429）

com.exceptioncoder.llm.application.task.LlmTaskService#getTask(String taskId): Optional<LlmTask>
  — cache.get → miss 则 repo.findById → put cache

com.exceptioncoder.llm.application.task.LlmTaskService#cancel(String taskId, String reason): void
  — repo.requestCancel；终态时抛 IllegalTaskStateException → API 转 409

com.exceptioncoder.llm.application.task.LlmTaskService#streamEvents(String taskId, long lastSeq): Flux<LlmTaskEvent>
  — 终态：仅 eventRepo.findByTaskIdAfterSeq → 发完自动 complete
  — 活跃：Flux.concat(eventRepo.findAfterSeq, publisher.subscribe).distinct(LlmTaskEvent::seq)

com.exceptioncoder.llm.application.task.LlmTaskQueryService#list(TaskQuery query): Page<LlmTask>
  — Spring Data Pageable，按 user/type/status 过滤

// === Infrastructure 层 ===
com.exceptioncoder.llm.infrastructure.task.repository.JpaLlmTaskRepository#save(LlmTask): LlmTask
  — Entity 转 Domain，调用 SpringDataRepo.save，写完 cache.put

com.exceptioncoder.llm.infrastructure.task.repository.JpaLlmTaskRepository#updateStatus(taskId, expected, next): boolean
  — Native UPDATE ... WHERE task_id=? AND status=? + @Modifying；rowsAffected>0 时 cache.put

com.exceptioncoder.llm.infrastructure.task.publisher.ReactorLlmTaskEventPublisher#publish(...): LlmTaskEvent
  — 1) 应用层 ReentrantLock per task 串行 seq 分配 2) UPDATE llm_task SET last_event_seq=last_event_seq+1 RETURNING 3) sinks.tryEmitNext 4) 终态：同步 appendBatch；过程：queue.offer

com.exceptioncoder.llm.infrastructure.task.publisher.EventFlushWorker#flush()
  — @Scheduled(fixedRate=1000)；取出 queue 当前所有 → appendBatch；失败记日志 + metrics

com.exceptioncoder.llm.infrastructure.task.publisher.ReactorLlmTaskEventPublisher#subscribe(taskId): Flux<LlmTaskEvent>
  — 每 task 一个 Sinks.Many.multicast().onBackpressureBuffer()；task 终态时调 sinks.tryEmitComplete + 移除 map entry

com.exceptioncoder.llm.infrastructure.task.cache.LlmTaskCache#put(LlmTask): void
  — 终态任务带 expireAfterWrite(retainMinutes)；活跃任务无过期

com.exceptioncoder.llm.infrastructure.task.pool.LlmTaskExecutorPool#submit(LlmTask task): Future<?>
  — 按 task.taskType 取对应 ExecutorService + Semaphore
  — runner: updateStatus(PENDING→RUNNING) → publish(STARTED) → executor.execute(task, ctx) → updateStatus(RUNNING→COMPLETED) → publish(COMPLETE)
  — 异常分支：publish(ERROR) + updateStatus(RUNNING→FAILED) + release semaphore

com.exceptioncoder.llm.infrastructure.task.pool.LlmTaskTimeoutWatcher#watch(future, taskId, timeoutSeconds)
  — daemon 线程 future.get(timeout, SECONDS)；超时 future.cancel(true) + publish(TIMEOUT) + updateStatus(RUNNING→TIMED_OUT)

com.exceptioncoder.llm.infrastructure.task.restore.LlmTaskRestoreRunner#run(ApplicationArguments)
  — repo.findInterrupted(1000) 分页
  — 每个 task：if retry<max → updateStatus(QUEUED/RUNNING→PENDING) + incrementRetry + publish(INTERRUPTED, "auto retry") + pool.submit
  — else → updateStatus→INTERRUPTED + publish(INTERRUPTED, "max retries reached")

com.exceptioncoder.llm.infrastructure.task.restore.LlmTaskCleanupScheduler#cleanup()
  — @Scheduled(cron="0 0 3 * * *") 每天凌晨 3 点
  — DELETE FROM llm_task_event WHERE task_id IN (SELECT task_id FROM llm_task WHERE finished_at < now() - INTERVAL ? DAY)
  — DELETE FROM llm_task WHERE finished_at < now() - INTERVAL ? DAY

// === API 层 ===
com.exceptioncoder.llm.api.controller.management.LlmTaskController#stream(taskId, lastEventId): SseEmitter
  — 取 Header Last-Event-ID（默认 0）
  — SseEmitter(sseTimeout) → service.streamEvents(taskId, lastSeq)
  — flux.subscribe(event → emitter.send(SseEmitter.event().id(seq).name(type.lower()).data(data)), err, () → emitter.send("[DONE]") + complete)

com.exceptioncoder.llm.api.controller.management.LlmTaskController#cancel(taskId, body): ResponseEntity
  — service.cancel → 202 Accepted；TASK_ALREADY_TERMINAL → 409
```

---

## 4. 数据结构

### 关键表及字段

```
表名：llm_task
字段：
  task_id VARCHAR(64) NOT NULL UNIQUE
  task_type VARCHAR(32) NOT NULL    // AGENT/DEVPLAN/CHAT/RESUME
  biz_ref VARCHAR(128)
  status VARCHAR(16) NOT NULL       // 见 LlmTaskStatus 枚举
  user_id VARCHAR(64)
  request_json TEXT
  result_json LONGTEXT
  error_message TEXT
  progress INT DEFAULT 0
  current_step VARCHAR(128)
  retry_count INT DEFAULT 0
  max_retries INT DEFAULT 0
  timeout_seconds INT DEFAULT 600
  cancellation_requested TINYINT DEFAULT 0
  last_event_seq BIGINT DEFAULT 0
  idempotency_key VARCHAR(128)
  trace_id VARCHAR(64)
  created_at DATETIME NOT NULL
  started_at DATETIME
  finished_at DATETIME
  expires_at DATETIME
索引：
  UNIQUE uk_task_id (task_id)
  UNIQUE uk_idempotency (idempotency_key)
  idx_status_created (status, created_at)
  idx_user_status (user_id, status, created_at)
  idx_type_status (task_type, status)

表名：llm_task_event
字段：
  task_id VARCHAR(64) NOT NULL
  event_seq BIGINT NOT NULL
  event_type VARCHAR(32) NOT NULL   // 见 LlmTaskEventType 枚举
  event_data LONGTEXT
  created_at DATETIME NOT NULL
索引：
  UNIQUE uk_task_seq (task_id, event_seq)
  idx_task_created (task_id, created_at)
```

### 关键 DTO 字段

```java
// LlmTaskSubmitCommand（domain 提交命令）
String taskType;            // 必填，决定路由 executor
String bizRef;              // 业务关联键
String userId;              // 当前用户
String requestJson;         // 请求快照
int maxRetries;             // 默认 0
int timeoutSeconds;         // 默认 600
String idempotencyKey;      // 可选幂等键
String traceId;             // 可选 trace 关联

// LlmTask（domain 实体 record）
String taskId; String taskType; String bizRef; LlmTaskStatus status;
String userId; String requestJson; String resultJson; String errorMessage;
int progress; String currentStep;
int retryCount; int maxRetries; int timeoutSeconds;
boolean cancellationRequested; long lastEventSeq;
String idempotencyKey; String traceId;
LocalDateTime createdAt; LocalDateTime startedAt; LocalDateTime finishedAt; LocalDateTime expiresAt;

// LlmTaskEvent（domain 实体 record）
String taskId; long seq; LlmTaskEventType type; String dataJson; LocalDateTime createdAt;
```

---

## 5. 重要约束与边界

- **幂等键**：`idempotency_key`，重复提交返回已存在 task；并发同 key 由 DB UNIQUE 约束兜底，捕获 `DataIntegrityViolationException` 转查询
- **并发控制**：按 `task_type` 分组的 `Semaphore`；超限抛 `ConcurrencyExceededException` → API 转 429
- **状态机锁**：`updateStatus` 用乐观锁 `WHERE status=expected`，rowsAffected=0 时表示状态已被其他线程变更，调用方决定重试或放弃
- **事件 seq 分配锁**：进程内 `ConcurrentHashMap<String, ReentrantLock>` 按 task_id 加锁串行化 DB 更新；分布式部署需改 Redis 原子递增（后续扩展）
- **事务范围**：
  - submit：`save(PENDING)` 在事务内；线程池 submit 在事务外（避免长事务）
  - executor 内部：单事件 publish 不嵌套大事务；终态变更 + 终态事件落盘可放同一事务内确保原子
- **不处理的场景**：
  - 跨进程任务（分布式调度）
  - token 级事件持久化与回放
  - SSE 服务端推送的反压（Reactor `onBackpressureBuffer` 默认 256，超出丢弃最旧 + 记 metrics）

---

## 6. 下游依赖调用

```
// 业务方实现 SPI
com.exceptioncoder.llm.infrastructure.agent.executor.AgentExecutor implements LlmTaskExecutor
  @LlmTaskExecutorType("AGENT")
  execute(task, ctx): 将 task.requestJson 反序列化为 AgentExecutionRequest，复用现有 ReAct 循环

com.exceptioncoder.llm.infrastructure.devplan.graph.DevPlanGraphExecutor implements LlmTaskExecutor
  @LlmTaskExecutorType("DEVPLAN")
  execute(task, ctx): 反序列化为 DevPlanSubmitCommand，调用现有 GraphExecutionEngine

// 现有 Spring Data JPA / Reactor Core，无新增外部依赖
```

---

## 7. 异常处理要点

- DB 写入冲突（`DataIntegrityViolationException`，idempotency_key 冲突）→ 转查询返回已存在任务
- 乐观锁失败（updateStatus 返回 false）→ 写日志 + metrics，业务侧决定重试或放弃，不抛异常
- 槽位满 → 抛 `ConcurrencyExceededException` → API 转 429
- 任务被取消 → 业务方在 `ctx.checkCancellation()` 处抛 `TaskCancelledException` → pool 捕获后 publish(CANCELLED) + updateStatus(RUNNING→CANCELLED)
- 超时 → watcher 抛 `TaskTimeoutException` → publish(TIMEOUT) + updateStatus(RUNNING→TIMED_OUT) + future.cancel(true)
- 事件批量 flush 失败 → 记 ERROR 日志 + 进 metrics（`llm_task_event_flush_failure_total`），不中断任务执行
- 热流推送失败（sinks 满）→ DEBUG 日志，丢弃最旧事件（onBackpressureBuffer 行为）
- 启动恢复失败 → 单 task 失败不影响其他，记 ERROR 日志，未恢复的 task 保留 RUNNING 状态等待人工介入
- 非任务所有者访问 → API 层校验 → 抛 `ForbiddenException` → GlobalExceptionHandler 转 403
- 任务不存在 → `TaskNotFoundException` → 404
- 已终态任务尝试取消 → `IllegalTaskStateException` → 409 `TASK_ALREADY_TERMINAL`
