# 编码摘要文档

> 本文档由完整设计文档精简而来，供 AI 辅助编码时使用，聚焦实现所需的最小必要信息。
> 对应完整文档：`平台管理接口补全-20260409-v1.md`
>
> **职责边界**：设计文档回答"哪些类、什么职责、怎么协作"，本文档回答"每个方法怎么写"。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-09 | 初始版本，4 个只读 GET 接口的编码细节 |

---

## 1. 核心业务规则

- 规则1：Graph 下 Agent 提取 — 遍历所有 `GraphNode`，提取 `config.get("agentId")` 非空的节点，去重后查 `AgentDefinitionRepository.findById()`，不存在则静默跳过
- 规则2：调用链遍历 — 从 `GraphDefinition.entryNodeId()` 出发深度优先遍历，`Set<String> visited` 防环，每步构建 `CallChainStep`
- 规则3：Agent Tool 解析 — `agent.toolIds()` 逐个调用 `ToolRegistry.getDefinition()`，`Optional.empty()` 静默跳过
- 规则4：Provider 聚合 — `findAllEnabled()` 后按 `LLMModelConfig.getProvider()` 分组计数

---

## 2. 接口契约

### 入口接口

```
GET /api/v1/graphs/{graphId}/agents
请求：路径变量 graphId: String
返回：List<AgentDefinition> — Graph 中所有关联 Agent 的完整定义

GET /api/v1/graphs/{graphId}/call-chain
请求：路径变量 graphId: String
返回：List<CallChainStep> — 按拓扑顺序排列的调用链步骤

GET /api/v1/agents/{agentId}/tools
请求：路径变量 agentId: String
返回：List<ToolDefinition> — Agent 关联的 Tool 完整定义

GET /api/v1/model-config/providers
请求：无参数
返回：List<ProviderInfo> — 各平台名称及模型数量
```

### 请求示例

```http
GET /api/v1/graphs/dev-plan-graph/agents
```

```http
GET /api/v1/graphs/dev-plan-graph/call-chain
```

```http
GET /api/v1/agents/code-awareness-agent/tools
```

```http
GET /api/v1/model-config/providers
```

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `c.e.l.application.usecase.GraphOrchestrationUseCase` | 修改 | 新增构造参数 `AgentDefinitionRepository`；新增 `getGraphAgents()`、`getGraphCallChain()`、`traverseChain()` |
| `c.e.l.application.usecase.GraphOrchestrationUseCase.CallChainStep` | 新增 | 调用链步骤 record |
| `c.e.l.application.usecase.AgentExecutionUseCase` | 修改 | 新增构造参数 `ToolRegistry`；新增 `getAgentTools()` |
| `c.e.l.application.usecase.LLMModelConfigUseCase` | 修改 | 新增 `getAllProviders()` |
| `c.e.l.application.usecase.LLMModelConfigUseCase.ProviderInfo` | 新增 | 平台信息 record |
| `c.e.l.api.controller.management.GraphController` | 修改 | 新增 2 个 GET 端点 |
| `c.e.l.api.controller.management.AgentController` | 修改 | 新增 1 个 GET 端点 |
| `c.e.l.api.controller.management.LLMModelConfigController` | 修改 | 新增 1 个 GET 端点 |

> `c.e.l` = `com.exceptioncoder.llm`

### 关键方法签名与职责

```
// ===== GraphOrchestrationUseCase =====

GraphOrchestrationUseCase#getGraphAgents(String graphId): List<AgentDefinition>
  — 遍历 Graph 所有 nodes，提取 config.agentId，去重查询 AgentDefinitionRepository

GraphOrchestrationUseCase#getGraphCallChain(String graphId): List<CallChainStep>
  — 从 entryNodeId 深度优先遍历，构建有序调用链

GraphOrchestrationUseCase#traverseChain(GraphDefinition graph, String nodeId, Set<String> visited, List<CallChainStep> chain): void
  — 递归遍历辅助方法，每个节点解析 agentId/agentName 并收集出边目标

// 新增 record
CallChainStep(int order, String nodeId, String nodeName, NodeType nodeType,
              String agentId, String agentName, List<String> nextNodeIds)

// ===== AgentExecutionUseCase =====

AgentExecutionUseCase#getAgentTools(String agentId): List<ToolDefinition>
  — 获取 Agent 后遍历 toolIds，逐个从 ToolRegistry.getDefinition() 查询，flatMap 跳过空值

// ===== LLMModelConfigUseCase =====

LLMModelConfigUseCase#getAllProviders(): List<ProviderInfo>
  — findAllEnabled() 后按 provider 分组计数

// 新增 record
ProviderInfo(String provider, int modelCount)

// ===== Controller 层（仅委托，无额外逻辑）=====

GraphController: @GetMapping("/{graphId}/agents") → graphOrchestrationUseCase.getGraphAgents(graphId)
GraphController: @GetMapping("/{graphId}/call-chain") → graphOrchestrationUseCase.getGraphCallChain(graphId)
AgentController: @GetMapping("/{agentId}/tools") → agentExecutionUseCase.getAgentTools(agentId)
LLMModelConfigController: @GetMapping("/providers") → modelConfigUseCase.getAllProviders()
```

---

## 4. 数据结构

### 关键 DTO/DO 字段

不涉及新增数据库表或字段。关键数据结构均为已有领域模型：

```java
// GraphNode — 已有 record
String id;
NodeType type;           // LLM, TOOL, CONDITION, MERGE, PARALLEL, LOOP, OUTPUT
String name;
Map<String, Object> config;  // config.get("agentId") 提取关联 Agent

// AgentDefinition — 已有 record
String id;
List<String> toolIds;    // 关联 Tool 的 ID 列表

// LLMModelConfig — 已有 class
String provider;         // 平台名称，用于分组
```

---

## 5. 重要约束与边界

- 幂等性：所有接口均为 GET 只读，天然幂等
- 并发控制：无需，不涉及写操作
- 事务范围：无需，纯查询
- 不处理的场景：Graph 中无 entryNodeId 时 `getGraphCallChain()` 返回空列表；agentId 关联的 Agent 已删除时 agentName 为 null 但不中断

---

## 6. 下游依赖调用

```
// 所有调用均为 Domain 层已有接口，无新增下游依赖
c.e.l.domain.repository.GraphDefinitionRepository#findById(String id): Optional<GraphDefinition>
c.e.l.domain.repository.AgentDefinitionRepository#findById(String id): Optional<AgentDefinition>
c.e.l.domain.registry.ToolRegistry#getDefinition(String toolId): Optional<ToolDefinition>
c.e.l.domain.repository.LLMModelConfigRepository#findAllEnabled(): List<LLMModelConfig>
```

---

## 7. 异常处理要点

- `graphId 不存在` → 抛出 `IllegalArgumentException("Graph 不存在: " + graphId)`，由 GlobalExceptionHandler 转 400
- `agentId 不存在` → 抛出 `IllegalArgumentException("Agent 不存在: " + agentId)`，由 GlobalExceptionHandler 转 400
- `toolId 在 ToolRegistry 中不存在` → `Optional.empty()`，`flatMap` 跳过，返回结果中不包含该 tool
- `Graph 节点 config 中无 agentId` → 该节点的 agentId/agentName 字段为 null，正常返回
