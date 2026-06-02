# 开发日志 2026-W15（2026-04-06 ~ 2026-04-06）

## 2026-04-06

### 任务描述
基于组件全景图 v4 架构实现「代码感知智能开发方案智能体」骨架代码，覆盖 Domain / Application / Infrastructure / API 四层，建立多 Agent 四角色协作的完整链路。

### 创建的文件

**设计文档** (`docs/design/代码感知智能开发方案智能体/`)：
- `代码感知智能开发方案智能体-20260406-v1.md` — 基于全景图 v3 的初始设计
- `代码感知智能开发方案智能体-20260406-v1-coding.md` — v1 编码摘要
- `代码感知智能开发方案智能体-20260406-v2.md` — 基于全景图 v4 重构，引入控制平面 + StateGraph + 多 Agent + Tool 标准协议 + 三级记忆 + 全链路 Trace
- `代码感知智能开发方案智能体-20260406-v2-coding.md` — v2 编码摘要

**Domain 层** (`llm-domain/domain/devplan/`)：
- `model/AgentRole.java` — Agent 角色枚举（4 角色）
- `model/DevPlanState.java` — OverAllState（跨 Node 共享全局状态）
- `model/DevPlanTask.java` — 任务实体
- `model/ProjectStructure.java` — 项目结构
- `model/ArchTopology.java` — 架构拓扑
- `model/RequirementIntent.java` — 需求意图
- `model/ImpactAnalysis.java` — 影响分析
- `model/DevPlanDocument.java` — 设计文档
- `model/ValidationResult.java` — 验证结果（含 PASS_THRESHOLD=70 业务规则）
- `model/CodeIndexStatus.java` — 索引状态
- `model/AgentOutput.java` — Agent 执行输出
- `service/DevPlanTaskManager.java` — 任务管理接口
- `service/DevPlanAgentRouter.java` — Agent 路由接口
- `service/PlanSensor.java` — 传感器接口
- `service/DevPlanMemoryManager.java` — 三级记忆管理接口
- `repository/DevPlanTaskRepository.java` — 任务仓储接口
- `repository/CodeIndexStatusRepository.java` — 索引状态仓储接口
- `repository/DevPlanRecordRepository.java` — 方案记录仓储接口
- `repository/ProjectArchTopologyRepository.java` — 架构拓扑仓储接口

**Application 层** (`llm-application/application/devplan/`)：
- `DevPlanFlowDefinition.java` — StateGraph 流程定义（4 Node + 修正循环）
- `ReviewRoutingStrategy.java` — 审查条件路由策略
- `node/ScanNode.java` — 代码感知节点编排
- `node/AnalyzeNode.java` — 需求分析节点编排
- `node/DesignNode.java` — 方案生成节点编排
- `node/ReviewNode.java` — 方案审查节点编排
- `usecase/DevPlanUseCase.java` — 顶层用例

**Infrastructure 层** (`llm-infrastructure/infrastructure/devplan/`)：
- `control/ConcurrencyController.java` — Semaphore 并发控制
- `control/TimeoutController.java` — Future.get(timeout) 超时控制
- `control/DevPlanTaskManagerImpl.java` — 任务管理实现
- `agent/DevPlanAgentRouterImpl.java` — Agent 路由实现
- `agent/DevPlanAgentConfig.java` — 各角色 Agent 配置（SystemPrompt）
- `sensor/ArchComplianceSensor.java` — 架构合规传感器
- `sensor/NamingConventionSensor.java` — 命名规范传感器
- `sensor/PlanSensorChain.java` — 传感器链
- `memory/DevPlanMemoryManagerImpl.java` — 三级记忆管理实现
- `repository/InMemoryDevPlanTaskRepository.java` — 任务仓储（内存实现）
- `repository/InMemoryProjectArchTopologyRepository.java` — 拓扑仓储（内存实现）

**API 层** (`llm-api/api/`)：
- `controller/DevPlanController.java` — REST 控制器（generate + task status）
- `dto/devplan/DevPlanRequest.java` — 请求 DTO
- `dto/devplan/DevPlanResponse.java` — 响应 DTO
- `dto/devplan/TaskStatusResponse.java` — 任务状态响应 DTO

### 修改的文件
- `docs/design/INDEX.md` — 追加「代码感知智能开发方案智能体」索引条目
- `docs/design/代码感知智能开发方案智能体/代码感知智能开发方案智能体-20260406-v2.md` — v2.1 修正 DDD 分层

### 关键设计决策
- **DDD 分层修正**：流程编排（StateGraph 节点定义 + 条件路由）属于用例编排逻辑，从 Infrastructure 移入 Application 层。Infrastructure 只保留技术实现（LLM 调用、文件系统、向量库、Redis 等）
- **依赖方向**：Application(Node) → Domain(接口) → Infrastructure(实现)，Node 调用 Domain 层 `DevPlanAgentRouter` 接口，由 Infrastructure 的 `DevPlanAgentRouterImpl` 注入实现
- **一期用内存仓储**：TaskRepository 和 TopologyRepository 使用 ConcurrentHashMap 内存实现，降低启动门槛，后续切换 JPA
- **传感器链设计**：通过 Spring `@Order` + `List<PlanSensor>` 自动注入实现执行顺序，计算型（ms 级）`@Order(1/2)` 先于推理型（s 级）
- **修正循环由 Application 层控制**：`DevPlanFlowDefinition.executeDesignReviewLoop()` 使用 while 循环 + `ReviewRoutingStrategy` 条件路由，而非 Infrastructure 层的技术框架控制
