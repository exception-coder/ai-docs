# 编码摘要文档

> 对应完整文档：`Agent异步执行机制-20260413-v2.md`
>
> **职责边界**：设计文档回答"哪些类、什么职责、怎么协作"，本文档回答"每个方法怎么写"。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-12 | 初始版本 |
| v2 | 2026-04-13 | AgentSseManager → AgentEventSink + Flux 模式；新增 AgentIterationEvent / AgentEventPublisher；UseCase 新增 streamExecution；Controller 消费 Flux |

---

## 1. 核心业务规则

- R1：`POST /execute` 必须在 50ms 内响应，禁止阻塞等待 LLM 调用
- R2：同时执行的 Agent 数量不超过 `agent.async.max-concurrent`（默认 5），超限返回 HTTP 429
- R3：单次执行总时长不超过 `AgentDefinition.timeoutSeconds`（默认 120s），超时 `Future.cancel(true)`
- R4：单次 LLM 调用超时后最多重试 2 次，3 次均失败则标记当前迭代失败
- R5：连续 3 轮迭代均因 LLM 超时失败，直接终止执行，状态 → FAILED
- R6：Flux 订阅者取消或 SSE 连接断开不影响后台 Agent 执行，结果保留在 AgentTaskStore
- R7：已完成/失败任务在 AgentTaskStore 中保留 30 分钟后自动清除
- R8：executionId 格式 `exec-{yyyyMMdd}-{自增序号}`
- R9：API 层不直接引用 Infrastructure 类，事件流通过 Domain 层 Flux 接口传递

---

## 2. 接口契约

### 入口接口

```
POST /api/v1/agents/{agentId}/execute（改造：同步 → 异步）
请求：AgentController.AgentExecuteRequest — input(String), context(Map)
返回：AgentExecuteAsyncResponse — executionId(String), agentId(String), status(String)
HTTP 状态码：202 Accepted

GET /api/v1/agents/executions/{executionId}（新增）
返回：AgentExecutionStatusResponse — executionId, agentId, status, currentIteration, maxIterations, finalOutput, errorMessage, thoughtHistory, toolCalls, elapsedMs

GET /api/v1/agents/executions/{executionId}/stream（新增，SSE）
返回：SseEmitter，内部订阅 Flux<AgentIterationEvent> 推送 iteration / tool_result / complete / error 事件
```

### 请求示例

```http
POST /api/v1/agents/devplan-requirement-analyzer/execute
Content-Type: application/json

{
  "input": "分析用户权限管理模块的需求影响范围",
  "context": {
    "projectPath": "/path/to/project",
    "requirement": "新增 RBAC 角色权限管理"
  }
}
```

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.llm.domain.model.AgentTask` | 新建 | Agent 执行任务生命周期 record |
| `com.exceptioncoder.llm.domain.model.AgentTaskManager` | 新建 | 任务生命周期管理接口 |
| `com.exceptioncoder.llm.domain.executor.AgentIterationListener` | 新建 | 迭代级事件回调接口（写入端） |
| `com.exceptioncoder.llm.domain.executor.AgentIterationEvent` | 新建 | 领域事件模型 record（v2 新增） |
| `com.exceptioncoder.llm.domain.executor.AgentEventPublisher` | 新建 | 事件流发布接口，返回 Flux（v2 新增） |
| `com.exceptioncoder.llm.domain.executor.AgentExecutor` | 改造 | 新增带 listener 参数的 execute 重载 |
| `com.exceptioncoder.llm.infrastructure.agent.task.AgentTaskManagerImpl` | 新建 | TaskManager 实现，含并发 + 超时控制 |
| `com.exceptioncoder.llm.infrastructure.agent.task.AgentTaskStore` | 新建 | ConcurrentHashMap 内存任务存储 |
| `com.exceptioncoder.llm.infrastructure.agent.task.AgentEventSink` | 新建 | 实现 Listener + Publisher，基于 Sinks.Many（v2 替代 AgentSseManager） |
| `com.exceptioncoder.llm.infrastructure.agent.task.AgentAsyncConfig` | 新建 | 线程池 + 并发参数配置类 |
| `com.exceptioncoder.llm.infrastructure.agent.executor.AlibabaAgentExecutor` | 改造 | ReAct 循环中回调 listener，改进异常处理 |
| `com.exceptioncoder.llm.application.usecase.AgentExecutionUseCase` | 改造 | 新增 submitAsync / getExecutionStatus / streamExecution |
| `com.exceptioncoder.llm.api.controller.management.AgentController` | 改造 | execute 改 202，新增 2 端点，SSE 端点消费 Flux |

### 关键方法签名与职责

```
// ─── Domain 层：AgentIterationEvent（v2 新增） ───
com.exceptioncoder.llm.domain.executor.AgentIterationEvent
  — record(executionId, type, iteration, data)
  — Type enum: ITERATION, TOOL_RESULT, COMPLETE, ERROR
  — 静态工厂：iteration(), toolResult(), complete(), error()

// ─── Domain 层：AgentEventPublisher（v2 新增） ───
com.exceptioncoder.llm.domain.executor.AgentEventPublisher#getEventStream(String executionId): Flux<AgentIterationEvent>
  — 获取指定执行任务的事件热流，多个订阅者共享

// ─── Domain 层：AgentTask ───
com.exceptioncoder.llm.domain.model.AgentTask
  — record(executionId, agentId, status, currentIteration, maxIterations,
           finalOutput, errorMessage, thoughtHistory, toolCalls,
           startedAt, completedAt, createdAt)
  — Status enum: SUBMITTED, RUNNING, COMPLETED, FAILED, TIMED_OUT
  — elapsedMs(): long — 计算已耗时
  — Builder + toBuilder 模式

// ─── Domain 层：AgentTaskManager ───
com.exceptioncoder.llm.domain.model.AgentTaskManager#submit(String agentId, String input, Map context, int timeoutSeconds): AgentTask
com.exceptioncoder.llm.domain.model.AgentTaskManager#getTask(String executionId): Optional<AgentTask>
com.exceptioncoder.llm.domain.model.AgentTaskManager#completeTask(String executionId, AgentExecutionResult result): void
com.exceptioncoder.llm.domain.model.AgentTaskManager#failTask(String executionId, String errorMessage, AgentTask.Status failStatus): void

// ─── Domain 层：AgentIterationListener ───
com.exceptioncoder.llm.domain.executor.AgentIterationListener#onIteration(String executionId, int iteration, String thought, ToolCall toolCall): void
com.exceptioncoder.llm.domain.executor.AgentIterationListener#onToolResult(String executionId, int iteration, String toolName, String output): void
com.exceptioncoder.llm.domain.executor.AgentIterationListener#onComplete(String executionId, AgentExecutionResult result): void
com.exceptioncoder.llm.domain.executor.AgentIterationListener#onError(String executionId, String errorMessage): void
com.exceptioncoder.llm.domain.executor.AgentIterationListener.NOOP — 空实现常量

// ─── Domain 层：AgentExecutor 改造 ───
com.exceptioncoder.llm.domain.executor.AgentExecutor#execute(AgentExecutionRequest request, AgentIterationListener listener): AgentExecutionResult
  — default 方法，委托给无参版本；实现类覆盖以支持回调

// ─── Infrastructure 层：AgentEventSink（v2 替代 AgentSseManager） ───
com.exceptioncoder.llm.infrastructure.agent.task.AgentEventSink
  — implements AgentIterationListener, AgentEventPublisher
  — ConcurrentHashMap<String, Sinks.Many<AgentIterationEvent>> sinks
  — getEventStream: computeIfAbsent 创建 Sinks.many().multicast().onBackpressureBuffer()
  — onIteration: toJson → AgentIterationEvent.iteration() → tryEmitNext
  — onToolResult: toJson → AgentIterationEvent.toolResult() → tryEmitNext
  — onComplete: tryEmitNext(complete event) → tryEmitComplete → remove sink
  — onError: tryEmitNext(error event) → tryEmitComplete → remove sink

// ─── Infrastructure 层：AgentTaskManagerImpl ───
  — 构造器注入 AgentIterationListener（领域接口，运行时注入 AgentEventSink）
  — submit: Semaphore.tryAcquire → 生成 executionId → save → threadPool.submit → startTimeoutWatcher
  — executeTask: 状态 RUNNING → executor.execute(request, listener) → completeTask/failTask → semaphore.release
  — startTimeoutWatcher: 虚拟线程 Future.get(timeout) → TimeoutException → cancel + failTask(TIMED_OUT)

// ─── Infrastructure 层：AgentTaskStore ───
  — save / get / update(UnaryOperator) / cleanExpired(@Scheduled)

// ─── Infrastructure 层：AlibabaAgentExecutor 改造 ───
  — callWithRetry: 单次 LLM 调用最多重试 2 次
  — consecutiveFailures 计数器，达到 3 时 break（R5 熔断）
  — notifyListener: try-catch 包装，回调异常不影响执行

// ─── Application 层：AgentExecutionUseCase 改造 ───
com.exceptioncoder.llm.application.usecase.AgentExecutionUseCase#submitAsync(String agentId, String input, Map context): AgentTask
com.exceptioncoder.llm.application.usecase.AgentExecutionUseCase#getExecutionStatus(String executionId): Optional<AgentTask>
com.exceptioncoder.llm.application.usecase.AgentExecutionUseCase#streamExecution(String executionId): Flux<AgentIterationEvent>
  — v2 新增，委托 agentEventPublisher.getEventStream()

// ─── API 层：AgentController 改造 ───
com.exceptioncoder.llm.api.controller.management.AgentController#execute: ResponseEntity（202 + AgentExecuteAsyncResponse）
com.exceptioncoder.llm.api.controller.management.AgentController#getExecutionStatus: ResponseEntity（轮询）
com.exceptioncoder.llm.api.controller.management.AgentController#streamExecution: SseEmitter
  — v2：创建 SseEmitter → UseCase.streamExecution(id) 获取 Flux → Flux.subscribe:
    - onNext: emitter.send(SseEmitter.event().name(type).data(json))
    - onError: emitter.completeWithError
    - onComplete: emitter.send("[DONE]") + emitter.complete()
  — 不再直接引用 AgentSseManager / AgentAsyncConfig 等 infrastructure 类
  — sseTimeout 通过 @Value("${agent.async.sse-timeout:600000}") 注入
```

---

## 4. 数据结构

### 关键 DTO/Record 字段

```java
// AgentIterationEvent（Domain record，v2 新增）
String executionId;         // 执行标识
Type type;                  // ITERATION / TOOL_RESULT / COMPLETE / ERROR
int iteration;              // 迭代轮次
String data;                // JSON 字符串（事件数据）

// AgentTask（Domain record）
String executionId;
String agentId;
Status status;              // SUBMITTED / RUNNING / COMPLETED / FAILED / TIMED_OUT
int currentIteration;
int maxIterations;
String finalOutput;
String errorMessage;
List<String> thoughtHistory;
List<ToolCall> toolCalls;
LocalDateTime startedAt;
LocalDateTime completedAt;
LocalDateTime createdAt;

// AgentExecuteAsyncResponse（API DTO）
String executionId;
String agentId;
String status;

// AgentExecutionStatusResponse（API DTO）
String executionId;
String agentId;
String status;
int currentIteration;
int maxIterations;
String finalOutput;
String errorMessage;
List<String> thoughtHistory;
List<ToolCall> toolCalls;
long elapsedMs;
```

### 配置结构

```yaml
agent:
  async:
    max-concurrent: 5
    thread-pool-size: 5
    execution-timeout: 120
    task-retain-minutes: 30
    sse-timeout: 600000
```

---

## 5. 重要约束与边界

- 并发控制：`Semaphore(maxConcurrent)` 非阻塞 `tryAcquire()`，失败立即返回，不排队等待
- 超时控制：`Future.get(timeoutSeconds, SECONDS)`，超时后 `future.cancel(true)` 中断执行线程
- Semaphore 释放保证：在 `finally` 块中释放，无论成功/失败/超时
- 内存存储限制：`AgentTaskStore` 使用 `ConcurrentHashMap`，服务重启后丢失，一期可接受
- 任务过期清理：`@Scheduled(fixedRate = 60000)` 每分钟扫描一次，清除超过 `taskRetainMinutes` 的已终态任务
- Flux 订阅取消不阻塞执行：SSE 断开仅取消 Flux 订阅，Sinks.Many 中后续事件仍可发布，不影响后台
- Sinks 自动清理：`onComplete` / `onError` 后从 `ConcurrentHashMap` 移除，防止内存泄漏
- 原同步接口保留：`AgentExecutionUseCase.execute()` 保持不变，供 DevPlan 等内部编排继续使用
- **分层约束（v2）**：`llm-api/pom.xml` 不依赖 `llm-infrastructure`；Controller 只通过 `UseCase` 获取 `Flux`
- 不处理的场景：多实例共享任务状态（需 Redis，二期）、任务优先级调度、执行结果持久化到数据库

---

## 6. 下游依赖调用

```
// LLM 调用（已有，不变）
com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter#route(String model): LLMProvider
com.exceptioncoder.llm.domain.service.LLMProvider#getChatModel(): ChatModel
org.springframework.ai.chat.model.ChatModel#call(Prompt): ChatResponse

// 工具执行（已有，不变）
com.exceptioncoder.llm.infrastructure.agent.tool.ToolExecutor#execute(String toolId, Map input): String

// Agent 定义查询（已有，不变）
com.exceptioncoder.llm.domain.repository.AgentDefinitionRepository#findById(String id): Optional<AgentDefinition>

// 事件流（v2 新增）
com.exceptioncoder.llm.domain.executor.AgentEventPublisher#getEventStream(String id): Flux<AgentIterationEvent>
```

---

## 7. 异常处理要点

- 并发超限 → `Semaphore.tryAcquire()` 返回 false → Controller 返回 `ResponseEntity.status(429).body(errorMsg)`
- Agent 不存在 → 抛出 `IllegalArgumentException` → Controller 返回 404
- 线程池拒绝 → 捕获 `RejectedExecutionException` → 释放 Semaphore → Controller 返回 503
- 整体超时 → `Future.get` 抛出 `TimeoutException` → `future.cancel(true)` → `AgentTaskStore.update(TIMED_OUT)` → `listener.onError()` → Flux complete
- 单次 LLM 超时 → `callWithRetry` 捕获异常 → `retryCount++`，不超过 2 次则重试
- 连续迭代失败 → `consecutiveFailures >= 3` → break 循环 → 状态 FAILED → Flux complete
- Sinks 发送失败 → `tryEmitNext` 返回 failure → 日志 DEBUG → 不影响执行
- SSE 发送失败 → Controller 中 Flux.subscribe 的 onNext 捕获 `IOException` → 日志 DEBUG → 不影响执行
