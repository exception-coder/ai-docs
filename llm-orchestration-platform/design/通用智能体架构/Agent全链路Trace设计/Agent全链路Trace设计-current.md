# Agent 全链路 Trace 设计

> 本文档定义平台级的 **Agent 全链路追踪（Trace）** 标准，适用于所有 Agent Group。
> 归属：通用智能体架构子模块
> 来源：从 `代码感知智能开发方案智能体/Tool层与Agent初始化器实现/Tool层与Agent初始化器实现-20260407-v2.md` 的 Trace 章节提取并通用化
> 关联文档：
> - 通用智能体架构 → `通用智能体架构-20260404-v1.md`（7 阶段执行模型）
> - 开发者指南 → `developer-guide/agent-development-guide.md`（Step 8: 可观测性接入）
> - 首个消费方 → devplan Agent Group

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|-------------|
| v1 | 2026-04-12 | zhangkai | 从 devplan Trace 设计提取为平台级通用设计；去除 DevPlan 前缀，定义为 AgentTraceRecorder |

---

## 1. 背景与目标

### 1.1 背景

通用智能体架构定义了 7 阶段执行模型（Input → Memory → Planning → Reasoning → Tool Selection → Tool Execution → Output），每个 Agent Group 的执行链路涉及多个 Node、Agent、Tool 的嵌套调用。当前缺乏统一的链路追踪机制，导致：

1. **调试困难**：无法定位某个 Agent 或 Tool 调用的耗时瓶颈
2. **可观测性缺失**：无法全局查看一次请求经过的完整调用路径
3. **各 Agent Group 自行实现**：如果不提供统一标准，每个 Group 会各自发明 Trace 方案，增加维护成本

### 1.2 目标

1. 定义平台级 Span 数据模型（`SpanContext`），描述一次调用的上下文
2. 提供轻量级 Trace 记录器（`AgentTraceRecorder`），支持 Span 嵌套和耗时统计
3. 一期基于内存 + SLF4J 日志，后续可平滑接入 OpenTelemetry
4. 所有 Agent Group 共用同一套 Trace 标准

### 1.3 设计边界

**本文档包含：**
- SpanContext 数据模型
- AgentTraceRecorder 组件设计
- 使用模式和接入指南

**本文档不包含：**
- OpenTelemetry 对接（二期）
- Trace 数据持久化到数据库（二期）
- 前端 Trace 可视化（独立需求）

---

## 2. 核心概念

| 概念 | 说明 |
|------|------|
| **Trace** | 一次完整请求的全链路记录，由唯一 `traceId` 标识 |
| **Span** | Trace 中的一个执行单元（如一个 Node、一次 Agent 调用、一次 Tool 执行） |
| **Parent Span** | 当前 Span 的父级，构成调用树 |

**调用树示例：**

```
Trace: task-abc-123
├── Span: ScanNode                        [1200ms]
│   ├── Span: devplan-code-awareness      [1100ms]
│   │   ├── Span: devplan_project_scan    [200ms]
│   │   ├── Span: devplan_dependency      [150ms]
│   │   ├── Span: devplan_code_structure  [400ms]
│   │   ├── Span: devplan_config_scan     [100ms]
│   │   └── Span: devplan_code_index      [250ms]
├── Span: AnalyzeNode                     [800ms]
│   └── Span: devplan-requirement-analyzer[780ms]
│       ├── Span: devplan_code_search     [300ms]
│       └── Span: devplan_file_read       [50ms]
├── Span: DesignNode                      [2000ms]
│   └── Span: devplan-solution-architect  [1950ms]
└── Span: ReviewNode                      [1500ms]
    └── Span: devplan-plan-reviewer       [1480ms]
```

---

## 3. 详细设计

### 3.1 SpanContext

```java
package com.exceptioncoder.llm.infrastructure.agent.trace;

/**
 * 一个执行单元的上下文信息。
 * 不可变 record，线程安全。
 */
public record SpanContext(
    String traceId,                    // 整个任务的 Trace ID
    String spanId,                     // 当前 Span ID（UUID）
    String parentSpanId,               // 父 Span ID（根 Span 为 null）
    String name,                       // Span 名称（如 "ScanNode", "devplan_project_scan"）
    long startTimeMs,                  // 开始时间戳（System.currentTimeMillis）
    Map<String, String> attributes     // 附加属性（如 projectPath, agentId, toolName）
) {}
```

**包路径变更说明：**
- 原路径：`c.e.l.infrastructure.devplan.trace.SpanContext`（devplan 专属）
- 新路径：`c.e.l.infrastructure.agent.trace.SpanContext`（平台通用）

### 3.2 AgentTraceRecorder

```java
package com.exceptioncoder.llm.infrastructure.agent.trace;

/**
 * 平台级 Agent 全链路追踪记录器。
 * 基于 ThreadLocal Span 栈实现嵌套 Span 管理。
 */
@Component
public class AgentTraceRecorder {

    // ── Trace 生命周期 ──

    /** 创建新 Trace，返回 traceId */
    String createTrace();

    /** 清理 Trace 数据（防止内存泄漏，任务结束时调用） */
    void cleanupTrace(String traceId);

    // ── Span 生命周期 ──

    /** 开始一个 Span，自动入栈并关联父 Span */
    SpanContext startSpan(String traceId, String name, Map<String, String> attributes);

    /** 结束 Span，出栈并记录耗时日志 */
    void endSpan(SpanContext span);

    // ── 查询 ──

    /** 获取某个 Trace 的全部 Span（按开始时间排序） */
    List<SpanContext> getSpans(String traceId);
}
```

**包路径变更说明：**
- 原路径：`c.e.l.infrastructure.devplan.trace.DevPlanTraceRecorder`
- 新路径：`c.e.l.infrastructure.agent.trace.AgentTraceRecorder`

### 3.3 内部实现

| 组件 | 实现方式 | 说明 |
|------|----------|------|
| Trace 存储 | `ConcurrentHashMap<String, List<SpanContext>>` | 按 traceId 索引所有 Span |
| Span 栈 | `ThreadLocal<Deque<SpanContext>>` | 支持 Span 嵌套，自动推导 parentSpanId |
| ID 生成 | `UUID.randomUUID().toString()` | traceId 和 spanId 均使用 UUID |
| 日志输出 | SLF4J 结构化日志 | 格式见下方 |

**日志格式：**

```
[TRACE] traceId={} spanId={} parentSpanId={} name={} elapsedMs={} attributes={}
```

示例：
```
[TRACE] traceId=abc-123 spanId=def-456 parentSpanId=null name=ScanNode elapsedMs=1200 attributes={projectPath=/opt/project}
[TRACE] traceId=abc-123 spanId=ghi-789 parentSpanId=def-456 name=devplan_project_scan elapsedMs=200 attributes={toolName=devplan_project_scan}
```

### 3.4 线程安全说明

- `SpanContext` 为不可变 record，天然线程安全
- `ConcurrentHashMap` 保证 Trace 存储的并发安全
- `ThreadLocal<Deque>` 保证同一线程内 Span 嵌套正确
- **跨线程场景**（如并行 Node 执行）：调用方需手动传递 traceId，在新线程中重新 startSpan

---

## 4. 使用模式

### 4.1 在 Node 中使用

```java
// 创建顶层 Trace
String traceId = traceRecorder.createTrace();

// 每个 Node 执行阶段创建 Span
var span = traceRecorder.startSpan(traceId, "ScanNode",
        Map.of("projectPath", projectPath));
try {
    AgentOutput output = router.route(AgentRole.CODE_AWARENESS, state);
    // ...
} finally {
    traceRecorder.endSpan(span);
}

// 任务结束清理（防止内存泄漏）
traceRecorder.cleanupTrace(traceId);
```

### 4.2 在 AgentExecutor 中自动埋点

AgentExecutor 可在 `execute()` 方法中自动创建 Agent 级 Span：

```java
var span = traceRecorder.startSpan(traceId, agentId,
        Map.of("role", role.name(), "iteration", String.valueOf(iteration)));
try {
    // ReAct loop...
} finally {
    traceRecorder.endSpan(span);
}
```

### 4.3 在 ToolExecutor 中自动埋点

ToolExecutor 可在 `execute()` 方法中自动创建 Tool 级 Span：

```java
var span = traceRecorder.startSpan(traceId, toolName,
        Map.of("toolName", toolName, "params", paramsSummary));
try {
    String result = toolMethod.invoke(bean, args);
    return result;
} finally {
    traceRecorder.endSpan(span);
}
```

> 完整的接入步骤参见 `developer-guide/agent-development-guide.md` 第 10 节（Step 8: 可观测性接入）。

---

## 5. 类清单（全类名）

| 全类名 | 操作 | 说明 |
|--------|------|------|
| `c.e.l.infrastructure.agent.trace.SpanContext` | 新建 | Record，不可变 Span 上下文 |
| `c.e.l.infrastructure.agent.trace.AgentTraceRecorder` | 新建 | @Component, ThreadLocal Span 栈 + ConcurrentHashMap 存储 |

> 注：`c.e.l` = `com.exceptioncoder.llm`

**迁移说明：** 原 devplan 专属的 `DevPlanTraceRecorder` 和 `SpanContext`（包路径 `c.e.l.infrastructure.devplan.trace.*`）应迁移到上述通用包路径。devplan 模块中的引用需同步更新 import。

---

## 6. 核心业务规则

| # | 规则 | 说明 |
|---|------|------|
| R1 | **Trace 全覆盖**：每个 Node / Agent / Tool 调用均生成 Span | 确保链路完整性 |
| R2 | Span 必须在 finally 块中 endSpan | 防止异常导致 Span 未关闭 |
| R3 | 任务结束必须 cleanupTrace | 防止内存泄漏 |
| R4 | Span name 使用有意义的标识 | Node 用类名，Agent 用 agentId，Tool 用 toolName |
| R5 | attributes 不记录敏感信息 | 不记录密码、token 等 |

---

## 7. 演进规划

| 阶段 | 内容 | 触发条件 |
|------|------|----------|
| **一期（当前）** | 内存 + SLF4J 日志 | devplan Agent Group 上线 |
| **二期** | 接入 OpenTelemetry SDK，导出到 Jaeger/Zipkin | 生产环境部署、需要持久化 Trace |
| **三期** | 前端 Trace Timeline 可视化 | Graph 可视化编排完成后扩展 |

**二期迁移路径：**
- `AgentTraceRecorder` 内部替换为 OTel `Tracer` + `Span`
- 外部 API（`createTrace` / `startSpan` / `endSpan`）保持不变
- 各 Agent Group 无需修改调用代码
