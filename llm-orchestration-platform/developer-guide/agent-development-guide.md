# 智能体开发指南

> 本文档面向需要在 llm-orchestration-platform 中开发新智能体（Agent Group）的开发者，
> 涵盖从角色定义到工具注册、编排接入、质量门禁的完整流程。

---

## 目录

1. [概念模型](#1-概念模型)
2. [开发前准备](#2-开发前准备)
3. [Step 1 -- 定义角色枚举](#3-step-1----定义角色枚举)
4. [Step 2 -- 开发 Tool](#4-step-2----开发-tool)
5. [Step 3 -- 配置 Agent](#5-step-3----配置-agent)
6. [Step 4 -- 编写 Agent 初始化器](#6-step-4----编写-agent-初始化器)
7. [Step 5 -- 实现 Agent Router](#7-step-5----实现-agent-router)
8. [Step 6 -- 接入 Graph 编排](#8-step-6----接入-graph-编排)
9. [Step 7 -- 添加质量门禁（可选）](#9-step-7----添加质量门禁可选)
10. [Step 8 -- 可观测性接入](#10-step-8----可观测性接入)
11. [启动顺序与生命周期](#11-启动顺序与生命周期)
12. [Checklist](#12-checklist)

---

## 1. 概念模型

```
┌─ Agent Group ──────────────────────────────────────────┐
│                                                        │
│  @AgentGroup(id="xxx")                                 │
│                                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│  │  Agent A  │───▶│  Agent B  │───▶│  Agent C  │        │
│  │ (Role A)  │    │ (Role B)  │    │ (Role C)  │        │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘         │
│       │               │               │                │
│  ┌────▼─────┐    ┌────▼─────┐    ┌────▼─────┐         │
│  │ Tool 1   │    │ Tool 3   │    │ Tool 5   │         │
│  │ Tool 2   │    │ Tool 4   │    │ Tool 6   │         │
│  └──────────┘    └──────────┘    └──────────┘         │
│                                                        │
│  State (immutable record, 各 Agent 读写)               │
└────────────────────────────────────────────────────────┘
```

| 概念 | 说明 |
|---|---|
| **Agent Group** | 一组协作 Agent 的编排单元，对应一个 `GraphDefinition` |
| **Agent Role** | 枚举值，标识 Agent 在流水线中的职责阶段 |
| **Tool** | Agent 可调用的原子能力，通过 `@Tool` 注解声明 |
| **State** | 不可变 record，贯穿流水线的全局状态快照 |
| **Sensor** | 质量门禁，对 Agent 输出做合规性检验 |

---

## 2. 开发前准备

### 分层约束

```
llm-domain          纯业务逻辑（枚举、record、接口），零框架依赖
llm-infrastructure  技术实现（Tool、Agent 路由、Sensor、初始化器）
llm-application     用例编排（UseCase，调 domain + infrastructure）
llm-api             REST 控制器
```

**规则：domain 不依赖 infrastructure；Tool / Initializer / Sensor 放 infrastructure。**

### 命名约定

| 类型 | 格式 | 示例 |
|---|---|---|
| Agent Group ID | `{group}` | `devplan`、`codereviewer` |
| Agent ID | `{group}-{role-kebab}` | `devplan-code-awareness` |
| Tool Name | `{group}_{tool_snake}` | `devplan_project_scan` |
| Role Enum | `UPPER_SNAKE_CASE` | `CODE_AWARENESS` |

---

## 3. Step 1 -- 定义角色枚举

在 `llm-domain` 中为你的智能体创建角色枚举。每个枚举值对应流水线中的一个阶段。

**路径：** `llm-domain/src/main/java/.../domain/{group}/model/{Group}Role.java`

```java
package com.exceptioncoder.llm.domain.mygroup.model;

/**
 * MyGroup 智能体角色枚举。
 */
public enum MyGroupRole {

    /** 第一阶段：数据采集 */
    DATA_COLLECTOR,

    /** 第二阶段：分析推理 */
    ANALYZER,

    /** 第三阶段：报告生成 */
    REPORTER
}
```

> **注意：** 如果你的 Tool 需要使用 `@Tool(roles = {...})` 进行角色绑定，
> 且复用了通用的 `@Tool` 注解（当前绑定了 `AgentRole` 类型），
> 你需要评估是否扩展 `@Tool.roles` 为通用机制，或为你的 Group 创建独立的 Tool 注解。
> 当前架构下，`@Tool.roles` 直接使用 `AgentRole` 枚举。

---

## 4. Step 2 -- 开发 Tool

Tool 是 Agent 的原子能力单元。通过注解声明，启动时自动注册。

**路径：** `llm-infrastructure/src/main/java/.../infrastructure/{group}/tool/{ToolName}Tool.java`

### 4.1 最小示例

```java
@Slf4j
@Component
public class DataFetchTool {

    @Tool(name = "mygroup_data_fetch",
          description = "从指定数据源拉取原始数据",
          tags = {"mygroup", "fetch"},
          roles = {AgentRole.DATA_COLLECTOR})       // 声明归属角色
    public String fetch(
            @ToolParam(value = "source", description = "数据源标识") String source,
            @ToolParam(value = "limit", description = "最大记录数",
                       required = false, defaultValue = "100") int limit) {
        // 实现逻辑...
        return jsonResult;
    }
}
```

### 4.2 注解说明

| 注解 | 字段 | 说明 |
|---|---|---|
| `@Tool` | `name` | 工具唯一 ID，格式 `{group}_{snake_name}` |
| | `description` | 供 LLM 理解的工具描述 |
| | `tags` | 分类标签，用于筛选 |
| | `roles` | `AgentRole[]` 枚举数组，声明哪些角色可调用此工具 |
| `@ToolParam` | `value` | 参数名 |
| | `description` | 供 LLM 理解的参数描述 |
| | `required` | 是否必填，默认 `true` |
| | `defaultValue` | 默认值（JSON 格式字符串） |

### 4.3 自动注册流程

```
启动 → ToolScanner 扫描所有 @Tool 方法
     → 反射提取参数，构建 JSON Schema
     → 读取 roles[]，转为 Set<String>
     → 注册 ToolDefinition 到 ToolRegistryImpl
```

**你不需要手动维护任何角色-工具映射表。** `@Tool(roles = {...})` 即声明即绑定。

### 4.4 开发规范

- 返回值统一为 `String`（JSON 格式），供 LLM 消费
- 失败时返回 `{"error": "..."}` 格式，不要抛异常到 LLM 侧
- Tool 内只做机械提取 / 计算，不做业务判断（判断交给 LLM）
- 依赖外部服务时，通过构造器注入 domain 层接口

---

## 5. Step 3 -- 配置 Agent

为每个角色配置 Agent ID 和 System Prompt。

**路径：** `llm-infrastructure/src/main/java/.../infrastructure/{group}/agent/{Group}AgentConfig.java`

```java
@Component
public class MyGroupAgentConfig {

    private static final Map<MyGroupRole, String> AGENT_IDS = Map.of(
            MyGroupRole.DATA_COLLECTOR, "mygroup-data-collector",
            MyGroupRole.ANALYZER,       "mygroup-analyzer",
            MyGroupRole.REPORTER,       "mygroup-reporter"
    );

    private static final Map<MyGroupRole, String> SYSTEM_PROMPTS = Map.of(
            MyGroupRole.DATA_COLLECTOR, """
                你是数据采集专家。你的职责是...
                ## 输出格式
                ...
                ## 约束
                ...
                """,
            MyGroupRole.ANALYZER, "...",
            MyGroupRole.REPORTER, "..."
    );

    public String getAgentId(MyGroupRole role) {
        return AGENT_IDS.get(role);
    }

    public String getSystemPrompt(MyGroupRole role) {
        return SYSTEM_PROMPTS.get(role);
    }
}
```

### System Prompt 编写要点

1. **角色定位**：明确身份和职责边界
2. **输出格式**：指定 JSON / Markdown 结构，方便下游 Agent 解析
3. **工具使用指引**：说明应在什么时机调用哪些 Tool
4. **约束条件**：明确禁止事项（如"不要编造数据"）

---

## 6. Step 4 -- 编写 Agent 初始化器

初始化器在应用启动时将 Agent 定义持久化到数据库，并声明 Graph 编排结构。

**路径：** `llm-infrastructure/src/main/java/.../infrastructure/{group}/config/{Group}AgentInitializer.java`

```java
@Slf4j
@Component
@Order(100)                                    // ToolScanner 之后
@AgentGroup(
    id = "mygroup",
    name = "我的智能体组",
    description = "xxx 场景的多 Agent 协作流水线"
)
public class MyGroupAgentInitializer
        implements ApplicationListener<ApplicationReadyEvent>, AgentGroupProvider {

    private final AgentDefinitionRepository agentRepository;
    private final MyGroupAgentConfig agentConfig;
    private final MyGroupToolRegistry toolRegistry;   // 角色-工具查询

    // 构造器注入...

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        for (MyGroupRole role : MyGroupRole.values()) {
            String agentId = agentConfig.getAgentId(role);
            if (agentRepository.existsById(agentId)) {
                log.info("Agent 已存在，跳过: {}", agentId);
                continue;
            }

            List<String> toolIds = toolRegistry.getToolsForRole(role)
                    .stream().map(ToolDefinition::id).toList();

            AgentDefinition agent = AgentDefinition.builder()
                    .id(agentId)
                    .name(role.name())
                    .systemPrompt(agentConfig.getSystemPrompt(role))
                    .toolIds(toolIds)
                    .maxIterations(10)
                    .timeoutSeconds(120)
                    .enabled(true)
                    .build();

            agentRepository.save(agent);
            log.info("Agent 初始化完成: {}, tools={}", agentId, toolIds);
        }
    }

    @Override
    public List<String> getAgentIds() {
        // 返回顺序即 Graph 中的节点执行顺序
        return Arrays.stream(MyGroupRole.values())
                .map(agentConfig::getAgentId)
                .toList();
    }
}
```

### 关键点

- `@Order(100)` 确保在 `ToolScanner`（默认 order）之后执行，否则 Tool 尚未注册
- `@AgentGroup` + `AgentGroupProvider` 会被 `AgentGroupScanner`（`@Order(200)`）扫描，自动生成 Graph
- `getAgentIds()` 返回的顺序决定了 Graph 中节点的串行执行顺序
- 幂等设计：`existsById` 检查，重启不会重复创建

---

## 7. Step 5 -- 实现 Agent Router

Router 负责在运行时构建每个角色的请求上下文并分派执行。

**域接口路径：** `llm-domain/src/main/java/.../domain/{group}/service/{Group}AgentRouter.java`

```java
public interface MyGroupAgentRouter {
    AgentOutput route(MyGroupRole role, MyGroupState state);
}
```

**实现路径：** `llm-infrastructure/src/main/java/.../infrastructure/{group}/agent/{Group}AgentRouterImpl.java`

```java
@Slf4j
@Component
public class MyGroupAgentRouterImpl implements MyGroupAgentRouter {

    private final AgentExecutor agentExecutor;
    private final MyGroupAgentConfig agentConfig;

    @Override
    public AgentOutput route(MyGroupRole role, MyGroupState state) {
        String agentId = agentConfig.getAgentId(role);
        String userInput = buildUserInput(role, state);

        var request = new AgentExecutor.AgentExecutionRequest(
                UUID.randomUUID().toString(),
                agentId,
                userInput,
                Map.of("taskId", state.taskId()),
                false
        );

        long start = System.currentTimeMillis();
        var result = agentExecutor.execute(request);
        long elapsed = System.currentTimeMillis() - start;

        return new AgentOutput(role.name(), result.finalOutput(),
                Map.of(), result.iterations(), elapsed);
    }

    private String buildUserInput(MyGroupRole role, MyGroupState state) {
        return switch (role) {
            case DATA_COLLECTOR -> "项目路径: " + state.projectPath();
            case ANALYZER -> "原始数据:\n" + state.rawData();
            case REPORTER -> "分析结果:\n" + state.analysis();
        };
    }
}
```

---

## 8. Step 6 -- 接入 Graph 编排

如果你使用了 `@AgentGroup` + `AgentGroupProvider`，Graph 会**自动生成**。

### 自动编排（默认）

`AgentGroupScanner` 在启动时：
1. 扫描所有 `@AgentGroup` 标注的 Bean
2. 调用 `getAgentIds()` 获取 Agent 列表
3. 为每个 Agent 创建 `GraphNode`（type = `LLM`）
4. 按列表顺序创建串行 `GraphEdge`
5. 注册 `GraphDefinition` 到仓储

```
Agent A ──▶ Agent B ──▶ Agent C    （自动生成的串行 Graph）
```

### 手动编排（高级）

如果需要条件分支、并行执行或循环，手动构建 `GraphDefinition`：

```java
GraphDefinition graph = GraphDefinition.builder()
    .id("mygroup")
    .name("带条件分支的流水线")
    .entryNodeId("collector")
    .nodes(List.of(
        new GraphNode("collector", NodeType.LLM, "数据采集", Map.of("agent", "mygroup-data-collector")),
        new GraphNode("check",    NodeType.CONDITION, "质量检查", Map.of()),
        new GraphNode("analyzer", NodeType.LLM, "分析", Map.of("agent", "mygroup-analyzer")),
        new GraphNode("retry",    NodeType.LLM, "重采集", Map.of("agent", "mygroup-data-collector")),
        new GraphNode("reporter", NodeType.LLM, "报告", Map.of("agent", "mygroup-reporter"))
    ))
    .edges(List.of(
        new GraphEdge("collector", "check", null),
        new GraphEdge("check", "analyzer", "quality >= 0.8"),
        new GraphEdge("check", "retry",    "quality < 0.8"),
        new GraphEdge("retry", "check",    null),
        new GraphEdge("analyzer", "reporter", null)
    ))
    .build();
```

支持的 `NodeType`：

| 类型 | 说明 |
|---|---|
| `LLM` | 调用 AgentExecutor 执行 LLM 推理 + Tool 调用 |
| `TOOL` | 直接调用单个 Tool（不经过 LLM） |
| `CONDITION` | 条件分支节点，根据上下文变量选择出边 |
| `PARALLEL` | 并行执行多个下游节点 |
| `LOOP` | 循环执行直到条件满足 |
| `OUTPUT` | 终止节点，输出最终结果 |

---

## 9. Step 7 -- 添加质量门禁（可选）

Sensor 对 Agent 输出做合规性检验，不通过时可触发修正循环。

**路径：** `llm-infrastructure/src/main/java/.../infrastructure/{group}/sensor/{SensorName}Sensor.java`

```java
@Component
@Order(1)    // 执行顺序
public class MyComplianceSensor implements PlanSensor {

    @Override
    public ValidationResult validate(DevPlanDocument document) {
        int score = 25;
        List<String> issues = new ArrayList<>();

        // 检查逻辑...
        if (!document.hasSection("architecture")) {
            score -= 10;
            issues.add("缺少架构设计章节");
        }

        return new ValidationResult(
                issues.isEmpty(),
                Math.max(score, 0),
                issues
        );
    }
}
```

`PlanSensorChain` 会自动聚合所有 `PlanSensor` Bean，按 `@Order` 顺序执行，累计分数和问题列表。

---

## 10. Step 8 -- 可观测性接入

使用 `DevPlanTraceRecorder` 记录执行链路。

```java
// 创建顶层 Trace
String traceId = traceRecorder.createTrace();

// 每个 Agent 执行阶段创建 Span
var span = traceRecorder.startSpan(traceId, "code-awareness",
        Map.of("projectPath", projectPath));
try {
    AgentOutput output = router.route(AgentRole.CODE_AWARENESS, state);
    // ...
} finally {
    traceRecorder.endSpan(span);
}

// 清理（防止内存泄漏）
traceRecorder.cleanupTrace(traceId);
```

每个 Span 自动记录耗时、父子关系，输出到 SLF4J 结构化日志。

---

## 11. 启动顺序与生命周期

```
ApplicationReadyEvent
    │
    ├── ToolScanner (@Order 默认)
    │     扫描 @Tool → 注册 ToolDefinition（含 roles）到 ToolRegistry
    │
    ├── {Group}AgentInitializer (@Order 100)
    │     创建 AgentDefinition → 持久化到 DB
    │     查询 ToolRegistry 获取角色绑定的 Tool 列表
    │
    └── AgentGroupScanner (@Order 200)
          扫描 @AgentGroup → 调用 getAgentIds()
          自动构建 GraphDefinition（节点 + 边）
```

**依赖关系：Tool 先注册 → Agent 初始化时读 Tool → Graph 最后组装。**

---

## 12. Checklist

开发一个新的智能体时，逐项确认：

### 定义层（domain）

- [ ] 创建角色枚举 `{Group}Role`
- [ ] 创建不可变状态 record `{Group}State`
- [ ] 定义 Router 接口 `{Group}AgentRouter`

### 实现层（infrastructure）

- [ ] 每个 Tool 类标注 `@Tool`，**roles 使用枚举引用**（不要硬编码字符串）
- [ ] 每个 Tool 参数标注 `@ToolParam`，描述清晰
- [ ] Tool 返回值为 JSON 字符串，失败返回 `{"error": "..."}`
- [ ] 创建 `{Group}AgentConfig`，配置 Agent ID 和 System Prompt
- [ ] 创建 `{Group}AgentInitializer`，标注 `@AgentGroup` + `@Order(100)`
- [ ] 实现 `AgentGroupProvider.getAgentIds()`，返回顺序即执行顺序
- [ ] 实现 `{Group}AgentRouterImpl`，构建角色特定的输入
- [ ] （可选）添加 Sensor 质量门禁

### 验证

- [ ] 启动日志确认 Tool 注册数量正确
- [ ] 启动日志确认 Agent 初始化成功
- [ ] 调用 `GET /api/v1/agents` 确认 Agent 可见
- [ ] 调用 `GET /api/v1/agents/{id}/tools` 确认 Tool 绑定正确
- [ ] 调用 `POST /api/v1/agents/{id}/execute` 端到端验证
