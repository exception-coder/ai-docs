# Agent 初始化器实现设计

> 本文档从「Tool 层、Trace 层与 Agent 初始化器实现设计 v2」中拆分而来，聚焦 **Agent 启动注册**。
> 父文档：`整体方案设计-20260406-v2.md`
> 关联文档：
> - Tool 层 → `Tool层实现设计-20260412-v1.md`（同目录）
> - Trace 全链路追踪 → `通用智能体架构/Agent全链路Trace设计/Agent全链路Trace设计-20260412-v1.md`
> - 开发者指南 → `developer-guide/agent-development-guide.md`（Step 4: 编写 Agent 初始化器）
> - 原合并文档 → `Tool层与Agent初始化器实现-20260407-v2.md`（归档，不再维护）

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|-------------|
| v1 | 2026-04-12 | zhangkai | 从合并文档拆分独立；内容与 v2 合并文档的 Agent 初始化部分一致 |

---

## 1. 背景与目标

### 1.1 背景

v2 整体框架已搭建完成，但数据库 `agent_definition` 表中无 `devplan-*` 记录，缺少类似 SecretaryAgentInitializer 的初始化器。调用链在 Agent 查找阶段断裂：

```
ScanNode.execute()
  → agentRouter.route(CODE_AWARENESS, state)
    → agentConfig.getAgentId() → "devplan-code-awareness"
      → agentExecutor.execute()
        → agentRepository.findById("devplan-code-awareness") → ❌ 空！
```

### 1.2 目标

实现 DevPlanAgentInitializer，应用启动时将 4 个角色 Agent 写入 `agent_definition` 表，使调用链完整贯通。

---

## 2. 详细设计

### 2.1 DevPlanAgentInitializer

**职责：** 应用启动时将 4 个 devplan 角色 Agent 写入数据库。

**类名：** `com.exceptioncoder.llm.infrastructure.devplan.config.DevPlanAgentInitializer`

**实现逻辑：**

```
监听 ApplicationReadyEvent（@Order 确保在 ToolScanner 之后执行）

for each AgentRole:
    1. agentId = DevPlanAgentConfig.getAgentId(role)
    2. if agentRepository.existsById(agentId) → 跳过（幂等）
    3. toolIds = DevPlanToolRegistry.ROLE_TOOL_MAPPING.get(role)
    4. systemPrompt = DevPlanAgentConfig.getSystemPrompt(role)
    5. 构建 AgentDefinition(id, name, description, systemPrompt, toolIds, ...)
    6. agentRepository.save(agentDefinition)
```

### 2.2 写入的 4 条 Agent 记录

| agentId | role | tool_ids |
|---------|------|----------|
| devplan-code-awareness | CODE_AWARENESS | ["devplan_project_scan", "devplan_dependency_analysis", "devplan_code_structure", "devplan_config_scan", "devplan_code_index"] |
| devplan-requirement-analyzer | REQUIREMENT_ANALYZER | ["devplan_code_search", "devplan_file_read"] |
| devplan-solution-architect | SOLUTION_ARCHITECT | ["devplan_code_search", "devplan_template_render"] |
| devplan-plan-reviewer | PLAN_REVIEWER | [] |

### 2.3 启动顺序依赖

```
ApplicationReadyEvent
    │
    ├── ToolScanner (@Order 默认)
    │     扫描 @Tool → 注册 ToolDefinition 到 ToolRegistry
    │
    ├── DevPlanAgentInitializer (@Order 100)
    │     创建 AgentDefinition → 持久化到 DB
    │     查询 ToolRegistry 获取角色绑定的 Tool 列表
    │
    └── AgentGroupScanner (@Order 200)
          扫描 @AgentGroup → 自动构建 GraphDefinition
```

**依赖关系：Tool 先注册 → Agent 初始化时读 Tool → Graph 最后组装。**

> 完整的启动顺序和开发步骤参见 `developer-guide/agent-development-guide.md` 第 11 节。

---

## 3. 类清单（全类名）

| 全类名 | 操作 | 说明 |
|--------|------|------|
| `c.e.l.infrastructure.devplan.config.DevPlanAgentInitializer` | 新建 | @Component, 启动时注册 4 个 Agent |

> 注：`c.e.l` = `com.exceptioncoder.llm`

**复用类：**

| 全类名 | 复用方式 |
|--------|----------|
| `c.e.l.domain.repository.AgentDefinitionRepository` | Agent 持久化 |
| `c.e.l.infrastructure.devplan.agent.DevPlanAgentConfig` | 角色→ID/Prompt 映射 |
| `c.e.l.infrastructure.devplan.tool.DevPlanToolRegistry` | 角色→工具集查询 |

---

## 4. 核心业务规则

| # | 规则 | 来源 |
|---|------|------|
| R1 | Agent 初始化幂等：existsById 已存在不覆盖 | 本文档 |
| R2 | Initializer 顺序：ToolScanner 先于 AgentInitializer（@Order 100） | 本文档 |
| R3 | AgentGroupScanner（@Order 200）在 AgentInitializer 之后，自动构建 Graph | 开发者指南 |

---

## 5. 异常处理

| 场景 | 处理方式 |
|------|----------|
| Agent 定义已存在 | DevPlanAgentInitializer 跳过，INFO 日志 |
| ToolRegistry 中工具尚未注册 | 启动失败（@Order 保证不应发生；若发生说明 Order 配置错误） |
| 数据库不可用 | 启动失败，Spring 标准异常处理 |

---

## 6. 测试要点

| 测试项 | 类型 | 说明 |
|--------|------|------|
| DevPlanAgentInitializer 幂等 | 集成测试 | 两次启动只创建一次 |
| Agent 定义完整性 | 集成测试 | 验证 4 条记录的 agentId、toolIds、systemPrompt 正确 |
| 启动顺序 | 集成测试 | 验证 Tool 注册先于 Agent 初始化 |
