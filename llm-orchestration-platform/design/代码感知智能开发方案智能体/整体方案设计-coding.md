# 编码摘要文档

> 本文档由完整设计文档精简而来，供 AI 辅助编码时使用。
> 对应完整文档：`整体方案设计-20260406-v2.md`

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-06 | 初始版本（基于 v3 全景图，单 Agent 四阶段） |
| v2 | 2026-04-06 | 基于 v4 全景图重构：控制平面 + StateGraph 编排 + 4 角色 Agent + Tool 标准协议 + 三级记忆 + 全链路 Trace |

---

## 1. 核心业务规则

- 规则1：并发控制 — 同时执行的方案生成任务不超过 `maxConcurrent`（默认 3），超限返回 429
- 规则2：超时控制 — 单任务总超时 300s，单 Node 超时 120s
- 规则3：Agent 角色隔离 — 每个 Agent 只能调用其角色对应的工具集，Tool Router 拒绝越权
- 规则4：工具 Schema 校验 — 所有 @Tool 调用前必须通过 JSON Schema 参数校验
- 规则5：索引去重 — 同 projectPath 只维护一个 Qdrant collection，基于 file_hash 判断是否重建
- 规则6：索引粒度 — 类级别，embedding = Javadoc + 类声明 + public 方法签名
- 规则7：架构分层 — api → application → domain → infrastructure，禁止反向依赖
- 规则8：命名强制 — Controller/UseCase/Repository/Impl 后缀
- 规则9：全类名必填 — 类设计所有类名含完整包路径
- 规则10：修正循环 — 审查不通过回退 DesignNode，最多 3 次
- 规则11：传感器顺序 — 计算型(ms)先于推理型(s)
- 规则12：静默成功/响亮失败 — 只将失败信息注入重试上下文
- 规则13：三级记忆 — 短期→Redis(TTL)，长期→Qdrant(永久)，结构化→MySQL(主动更新)
- 规则14：Trace 全覆盖 — 每个 Node/LLM/Tool 调用均生成 Span

---

## 2. 接口契约

### 入口接口

```
POST /v1/dev-plan/generate
请求：com.exceptioncoder.llm.api.dto.devplan.DevPlanRequest - projectPath, requirement, templateType, forceReindex, timeoutSeconds
返回：com.exceptioncoder.llm.api.dto.devplan.DevPlanResponse - taskId, document, impactAnalysis, validationResult, metadata

POST /v1/dev-plan/generate/stream
请求：同上
返回：SSE 流，按 Node 推送进度事件

POST /v1/dev-plan/index
请求：com.exceptioncoder.llm.api.dto.devplan.CodeIndexRequest - projectPath, includePatterns, excludePatterns
返回：CodeIndexStatusResponse

GET /v1/dev-plan/index/status?projectPath={path}
返回：CodeIndexStatusResponse

GET /v1/dev-plan/task/{taskId}
返回：com.exceptioncoder.llm.api.dto.devplan.TaskStatusResponse - taskId, status, currentNode, progress, elapsedMs
```

### 请求示例

```http
POST /v1/dev-plan/generate
Content-Type: application/json

{
  "projectPath": "/Users/zhangkai/IdeaProjects/llm-orchestration-platform",
  "requirement": "新增简历解析功能",
  "templateType": "STANDARD",
  "forceReindex": false,
  "timeoutSeconds": 300
}
```

### 关键方法签名（全类名）

```java
// API 层
com.exceptioncoder.llm.api.controller.DevPlanController#generateDevPlan(DevPlanRequest): ResponseEntity<DevPlanResponse>
com.exceptioncoder.llm.api.controller.DevPlanController#generateDevPlanStream(DevPlanRequest): SseEmitter
com.exceptioncoder.llm.api.controller.DevPlanController#getTaskStatus(String taskId): ResponseEntity<TaskStatusResponse>

// Application 层
com.exceptioncoder.llm.application.usecase.DevPlanUseCase#generateDevPlan(DevPlanRequest): DevPlanResponse

// 控制平面
com.exceptioncoder.llm.infrastructure.devplan.control.DevPlanTaskManagerImpl#submitTask(DevPlanRequest): DevPlanTask
com.exceptioncoder.llm.infrastructure.devplan.control.DevPlanTaskManagerImpl#completeTask(String taskId, DevPlanResult): void
com.exceptioncoder.llm.infrastructure.devplan.control.ConcurrencyController#acquire(): void
com.exceptioncoder.llm.infrastructure.devplan.control.ConcurrencyController#release(): void
com.exceptioncoder.llm.infrastructure.devplan.control.TimeoutController#executeWithTimeout(Callable<T>, int seconds): T

// 流程编排
com.exceptioncoder.llm.infrastructure.devplan.flow.DevPlanFlowEngineImpl#execute(DevPlanState): DevPlanResult
com.exceptioncoder.llm.infrastructure.devplan.flow.DevPlanStateGraph#build(): StateGraph<DevPlanState>
com.exceptioncoder.llm.infrastructure.devplan.flow.ScanNode#execute(DevPlanState): DevPlanState
com.exceptioncoder.llm.infrastructure.devplan.flow.AnalyzeNode#execute(DevPlanState): DevPlanState
com.exceptioncoder.llm.infrastructure.devplan.flow.DesignNode#execute(DevPlanState): DevPlanState
com.exceptioncoder.llm.infrastructure.devplan.flow.ReviewNode#execute(DevPlanState): DevPlanState
com.exceptioncoder.llm.infrastructure.devplan.flow.ReviewConditionalEdge#route(DevPlanState): String

// Agent 路由
com.exceptioncoder.llm.infrastructure.devplan.agent.DevPlanAgentRouterImpl#route(AgentRole, DevPlanState): AgentOutput

// 角色 Agent
com.exceptioncoder.llm.infrastructure.devplan.agent.CodeAwarenessAgent#execute(DevPlanState): AgentOutput
com.exceptioncoder.llm.infrastructure.devplan.agent.RequirementAnalyzerAgent#execute(DevPlanState): AgentOutput
com.exceptioncoder.llm.infrastructure.devplan.agent.SolutionArchitectAgent#execute(DevPlanState): AgentOutput
com.exceptioncoder.llm.infrastructure.devplan.agent.PlanReviewerAgent#execute(DevPlanState): AgentOutput

// 工具
com.exceptioncoder.llm.infrastructure.devplan.tool.ProjectScanTool#scan(String projectPath): ProjectStructure
com.exceptioncoder.llm.infrastructure.devplan.tool.CodeIndexTool#indexIfNeeded(String projectPath, boolean force): CodeIndexStatus
com.exceptioncoder.llm.infrastructure.devplan.tool.ArchTopologyTool#extractTopology(String projectPath): ArchTopology
com.exceptioncoder.llm.infrastructure.devplan.tool.CodeSearchTool#search(String query, int topK): List<RelevantCode>
com.exceptioncoder.llm.infrastructure.devplan.tool.FileReadTool#readFile(String filePath): String
com.exceptioncoder.llm.infrastructure.devplan.tool.TemplateRenderTool#render(String template, Map<String,Object> context): String
com.exceptioncoder.llm.infrastructure.devplan.tool.DevPlanToolRegistry#getToolsForRole(AgentRole): List<ToolDefinition>
com.exceptioncoder.llm.infrastructure.devplan.tool.DevPlanToolRegistry#validateAndExecute(String toolName, Map params): Object

// 传感器
com.exceptioncoder.llm.infrastructure.devplan.sensor.PlanSensorChain#validate(DevPlanDocument): ValidationResult
com.exceptioncoder.llm.infrastructure.devplan.sensor.ArchComplianceSensor#check(DevPlanDocument): SensorResult
com.exceptioncoder.llm.infrastructure.devplan.sensor.NamingConventionSensor#check(DevPlanDocument): SensorResult
com.exceptioncoder.llm.infrastructure.devplan.sensor.LlmJudgeSensor#check(DevPlanDocument): SensorResult

// 三级记忆
com.exceptioncoder.llm.infrastructure.devplan.memory.DevPlanMemoryManagerImpl#loadContext(String taskId): MemoryContext
com.exceptioncoder.llm.infrastructure.devplan.memory.DevPlanMemoryManagerImpl#persist(String taskId, DevPlanState): void
com.exceptioncoder.llm.infrastructure.devplan.memory.ShortTermMemoryStore#getRecentMessages(String taskId, int window): List<Message>
com.exceptioncoder.llm.infrastructure.devplan.memory.LongTermMemoryStore#searchRelevant(String query, int topK): List<VectorSearchResult>
com.exceptioncoder.llm.infrastructure.devplan.memory.StructuredMemoryStore#getTopology(String projectPath): ArchTopology

// Trace
com.exceptioncoder.llm.infrastructure.devplan.trace.DevPlanTraceRecorder#startSpan(String name): SpanContext
com.exceptioncoder.llm.infrastructure.devplan.trace.DevPlanTraceRecorder#endSpan(SpanContext): void
```

---

## 3. 涉及类清单（全类名）

| 全类名 | 操作 | 说明 |
|--------|------|------|
| **API 层** | | |
| `com.exceptioncoder.llm.api.controller.DevPlanController` | 新建 | 5 个端点 |
| `com.exceptioncoder.llm.api.dto.devplan.DevPlanRequest` | 新建 | |
| `com.exceptioncoder.llm.api.dto.devplan.DevPlanResponse` | 新建 | |
| `com.exceptioncoder.llm.api.dto.devplan.CodeIndexRequest` | 新建 | |
| `com.exceptioncoder.llm.api.dto.devplan.CodeIndexStatusResponse` | 新建 | |
| `com.exceptioncoder.llm.api.dto.devplan.TaskStatusResponse` | 新建 | |
| **Application 层** | | |
| `com.exceptioncoder.llm.application.usecase.DevPlanUseCase` | 新建 | 编排用例 |
| **Domain 层** | | |
| `com.exceptioncoder.llm.domain.devplan.model.DevPlanTask` | 新建 | 任务实体 |
| `com.exceptioncoder.llm.domain.devplan.model.DevPlanState` | 新建 | OverAllState |
| `com.exceptioncoder.llm.domain.devplan.model.ProjectStructure` | 新建 | |
| `com.exceptioncoder.llm.domain.devplan.model.ArchTopology` | 新建 | |
| `com.exceptioncoder.llm.domain.devplan.model.RequirementIntent` | 新建 | |
| `com.exceptioncoder.llm.domain.devplan.model.ImpactAnalysis` | 新建 | |
| `com.exceptioncoder.llm.domain.devplan.model.DevPlanDocument` | 新建 | |
| `com.exceptioncoder.llm.domain.devplan.model.ValidationResult` | 新建 | |
| `com.exceptioncoder.llm.domain.devplan.model.CodeIndexStatus` | 新建 | |
| `com.exceptioncoder.llm.domain.devplan.model.AgentRole` | 新建 | 枚举 |
| `com.exceptioncoder.llm.domain.devplan.service.DevPlanTaskManager` | 新建 | 接口 |
| `com.exceptioncoder.llm.domain.devplan.service.DevPlanFlowEngine` | 新建 | 接口 |
| `com.exceptioncoder.llm.domain.devplan.service.DevPlanAgentRouter` | 新建 | 接口 |
| `com.exceptioncoder.llm.domain.devplan.service.PlanSensor` | 新建 | 接口 |
| `com.exceptioncoder.llm.domain.devplan.service.DevPlanMemoryManager` | 新建 | 接口 |
| **Infrastructure - 控制平面** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.control.DevPlanTaskManagerImpl` | 新建 | 任务生命周期 |
| `com.exceptioncoder.llm.infrastructure.devplan.control.ConcurrencyController` | 新建 | Semaphore |
| `com.exceptioncoder.llm.infrastructure.devplan.control.TimeoutController` | 新建 | TimeLimiter |
| **Infrastructure - 流程编排** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.flow.DevPlanFlowEngineImpl` | 新建 | StateGraph 执行 |
| `com.exceptioncoder.llm.infrastructure.devplan.flow.DevPlanStateGraph` | 新建 | 图定义 |
| `com.exceptioncoder.llm.infrastructure.devplan.flow.ScanNode` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.flow.AnalyzeNode` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.flow.DesignNode` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.flow.ReviewNode` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.flow.ReviewConditionalEdge` | 新建 | 条件路由 |
| **Infrastructure - Agent** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.agent.DevPlanAgentRouterImpl` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.agent.CodeAwarenessAgent` | 新建 | 代码感知角色 |
| `com.exceptioncoder.llm.infrastructure.devplan.agent.RequirementAnalyzerAgent` | 新建 | 需求分析角色 |
| `com.exceptioncoder.llm.infrastructure.devplan.agent.SolutionArchitectAgent` | 新建 | 方案生成角色 |
| `com.exceptioncoder.llm.infrastructure.devplan.agent.PlanReviewerAgent` | 新建 | 方案审查角色 |
| `com.exceptioncoder.llm.infrastructure.devplan.agent.DevPlanAgentConfig` | 新建 | Agent 配置 |
| **Infrastructure - 工具** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.tool.ProjectScanTool` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.tool.CodeIndexTool` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.tool.ArchTopologyTool` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.tool.CodeSearchTool` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.tool.FileReadTool` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.tool.TemplateRenderTool` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.tool.DevPlanToolRegistry` | 新建 | Tool 标准协议 |
| **Infrastructure - 传感器** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.sensor.ArchComplianceSensor` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.sensor.NamingConventionSensor` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.sensor.LlmJudgeSensor` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.sensor.PlanSensorChain` | 新建 | |
| **Infrastructure - 记忆** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.memory.DevPlanMemoryManagerImpl` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.memory.ShortTermMemoryStore` | 新建 | Redis |
| `com.exceptioncoder.llm.infrastructure.devplan.memory.LongTermMemoryStore` | 新建 | Qdrant |
| `com.exceptioncoder.llm.infrastructure.devplan.memory.StructuredMemoryStore` | 新建 | MySQL |
| **Infrastructure - 观测** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.trace.DevPlanTraceRecorder` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.trace.SpanContext` | 新建 | Record |
| **Infrastructure - 持久化** | | |
| `com.exceptioncoder.llm.infrastructure.devplan.repository.CodeIndexStatusRepository` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.repository.DevPlanRecordRepository` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.repository.DevPlanTaskRepository` | 新建 | |
| `com.exceptioncoder.llm.infrastructure.devplan.repository.ProjectArchTopologyRepository` | 新建 | |
| **复用类** | | |
| `com.exceptioncoder.llm.domain.executor.AgentExecutor` | 复用 | |
| `com.exceptioncoder.llm.infrastructure.agent.executor.AlibabaAgentExecutor` | 复用 | |
| `com.exceptioncoder.llm.infrastructure.agent.tool.ToolRegistryImpl` | 复用 | |
| `com.exceptioncoder.llm.infrastructure.agent.tool.ToolScanner` | 复用 | |
| `com.exceptioncoder.llm.domain.service.LLMProvider` | 复用 | |
| `com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter` | 复用 | |
| `com.exceptioncoder.llm.domain.repository.VectorStoreRepository` | 复用 | |
| `com.exceptioncoder.llm.infrastructure.vector.QdrantVectorStoreRepository` | 复用 | |

---

## 4. 数据结构

### 关键表及字段

```sql
-- 任务表
CREATE TABLE dev_plan_task (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  task_id VARCHAR(50) NOT NULL,
  project_path VARCHAR(500) NOT NULL,
  requirement TEXT NOT NULL,
  status VARCHAR(20) NOT NULL COMMENT 'CREATED/QUEUED/RUNNING/COMPLETED/FAILED/TIMED_OUT',
  current_node VARCHAR(50),
  priority INT NOT NULL DEFAULT 0,
  timeout_seconds INT NOT NULL DEFAULT 300,
  trace_id VARCHAR(100),
  started_at DATETIME,
  completed_at DATETIME,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  UNIQUE INDEX uk_task_id (task_id),
  INDEX idx_status (status),
  INDEX idx_created_at (created_at)
);

-- 代码索引状态表
CREATE TABLE code_index_status (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  project_path VARCHAR(500) NOT NULL,
  collection_name VARCHAR(100) NOT NULL,
  doc_count INT NOT NULL DEFAULT 0,
  status VARCHAR(20) NOT NULL COMMENT 'IDLE/INDEXING/READY/FAILED',
  last_indexed_at DATETIME,
  file_hash VARCHAR(64),
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE INDEX uk_project_path (project_path)
);

-- 方案记录表
CREATE TABLE dev_plan_record (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  task_id VARCHAR(50) NOT NULL,
  project_path VARCHAR(500) NOT NULL,
  requirement TEXT NOT NULL,
  document LONGTEXT,
  impact_analysis JSON,
  validation_score INT,
  validation_issues JSON,
  node_timings JSON,
  agent_token_usage JSON,
  correction_count INT DEFAULT 0,
  model_used VARCHAR(50),
  total_token_usage INT,
  elapsed_ms BIGINT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_task_id (task_id),
  INDEX idx_project_path (project_path)
);

-- 结构化记忆：项目架构拓扑
CREATE TABLE project_arch_topology (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  project_path VARCHAR(500) NOT NULL,
  topology JSON NOT NULL,
  module_list JSON,
  layer_dependencies JSON,
  last_analyzed_at DATETIME,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE INDEX uk_project_path (project_path)
);
```

### 关键 Record 字段

```java
// DevPlanState（OverAllState — 跨 Node 共享）
String taskId;
String projectPath;
String requirement;
ProjectStructure structure;       // ScanNode 写入
ArchTopology topology;            // ScanNode 写入
RequirementIntent intent;         // AnalyzeNode 写入
ImpactAnalysis impact;            // AnalyzeNode 写入
DevPlanDocument document;         // DesignNode 写入
ValidationResult validation;     // ReviewNode 写入
List<String> reviewIssues;        // ReviewNode 写入（修正循环用）
int correctionCount;              // ReviewNode 递增
String status;                    // 各 Node 更新

// AgentRole（枚举）
CODE_AWARENESS, REQUIREMENT_ANALYZER, SOLUTION_ARCHITECT, PLAN_REVIEWER

// DevPlanTask
String taskId;
String status;     // CREATED/QUEUED/RUNNING/COMPLETED/FAILED/TIMED_OUT
String currentNode;
int priority;
int timeoutSeconds;
String traceId;
```

---

## 5. 重要约束与边界

- 并发控制：Semaphore(maxConcurrent=3)，acquire 在任务开始前，release 在任务完成后
- 超时控制：Future.get(timeout) 或 Resilience4j TimeLimiter
- 角色工具映射：CODE_AWARENESS→{ProjectScan,CodeIndex,ArchTopology}，REQUIREMENT_ANALYZER→{CodeSearch,FileRead}，SOLUTION_ARCHITECT→{CodeSearch,TemplateRender}，PLAN_REVIEWER→{无外部工具}
- StateGraph 条件路由：ReviewNode 输出 score≥70→END，score<70且count<3→DesignNode，否则→END(with_issues)
- 三级记忆写入：短期→Redis(TTL 1h)，长期→Qdrant(永久)，结构化→MySQL(按 projectPath UPSERT)
- 不处理：代码生成、多项目联合、实时监听、人工审批工作流
- 降级：向量库不可用→纯 LLM 分析，标注降级

---

## 6. 下游依赖调用

```java
com.exceptioncoder.llm.domain.service.LLMProvider#chat(LLMRequest): LLMResponse
com.exceptioncoder.llm.domain.service.LLMProvider#chatStream(LLMRequest): Flux<String>
com.exceptioncoder.llm.domain.repository.VectorStoreRepository#store(Document): void
com.exceptioncoder.llm.domain.repository.VectorStoreRepository#search(VectorSearchRequest): List<VectorSearchResult>
com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter#route(String model): LLMProvider
// Redis: Lettuce — ShortTermMemoryStore
// Qdrant: QdrantVectorStoreRepository — LongTermMemoryStore
// MySQL: JPA — StructuredMemoryStore, TaskRepository, RecordRepository
```

---

## 7. 异常处理要点

- 路径不存在 → `IllegalArgumentException` / 40001
- 无 Java 源文件 → `UnsupportedProjectException` / 40003
- 并发超限 → `ConcurrencyExceededException`（新建）/ 42901
- 任务超时 → `TaskTimeoutException`（新建）/ 50002
- LLM 不可用 → 重试 2 次 → `LLMProviderException` / 50001
- 向量库不可用 → 降级无 RAG，WARN 日志
- 工具权限拒绝 → `ToolPermissionDeniedException`（新建）/ 跳过工具
