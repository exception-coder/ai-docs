# llm-orchestration-platform / design 文档索引（用户目录知识库）

> 本目录承载 AI 生成的设计文档与长期演进版本。文件名一律 `-current.md`，配套 coding 摘要为 `-coding.md`；不带日期、不带版本号，历史变更靠 git log。

## 聊天回答一键复制
- 文件：`聊天回答一键复制/聊天回答一键复制-current.md`
- 摘要：前端聊天页 `Chat.tsx` assistant 消息底部新增"复制回答"按钮，复制 `msg.content` 原文（Markdown 源），优先 navigator.clipboard、移动端非安全上下文降级 textarea+execCommand，1.5s "已复制"反馈。纯前端单文件改动，无后端变更。轻量模版，无 coding.md。
- 大纲：代码入口 / 交互契约 / 核心流程图 / 关键规则 / 失败行为 / 升级触发条件

## 岗位技能趋势聚合
- 文件：`岗位技能趋势聚合/岗位技能趋势聚合-current.md` · [api](岗位技能趋势聚合/岗位技能趋势聚合-api-current.md) · [coding](岗位技能趋势聚合/岗位技能趋势聚合-coding.md)
- 摘要：新增 `skilltrend` 子系统。入库路径增挂"岗位族归一(自由填+别名映射表,未命中调LLM回写)+JD元数据落库(含 rawText)"；聚合按(标准岗位族,日期)从关系表枚举 JD,LLM **一次10条批量**抽取归一技能词并推断级别(SeniorityLevel),按(family,level)聚合词频/占比并按可靠阈值筛"主流可靠技能",每日生成快照;异步复用 llm-task-management。新增 4 表:job_posting_meta / job_family_alias / skill_trend_snapshot / skill_trend_item。
- 大纲：目标与边界(含4决策表)/ 整体架构图 / 模块拆分(6)/ 关键交互(入库归一+批量聚合+查询 3 图)/ 核心业务规则(R1-R11)/ 编码落点(四模块树)/ 数据与依赖变更(4表)/ 风险待确认 / 验证要点
- 接口：POST /aggregate（异步触发）· GET /daily（当日榜单）· GET /trend（历史趋势）

## 个人简历优化智能体
- 文件：`个人简历优化智能体/个人简历优化智能体-current.md`
- 摘要：为 kai-toolbox 简历模块提供 WORK / PROJECT / SELF_INTRO 三段单步 AI 改写能力，强制以目标岗位 JD 为锚点对齐硬技能关键词，输出结构化 JSON 让前端做 diff 预览。架构上仅复用 LLMOrchestrationService 单步 ChatModel 调用，不引入 Tool / Graph 编排。
- 大纲：背景与目标 / 功能范围 / 业务流程（同步+SSE 流式+异常+状态流转）/ 接口设计（2 端点）/ 类设计（10 类，含 ResumePromptProvider SPI）/ Prompt 设计要点（3 套）/ 前端集成（kai-toolbox optimize 子模块）/ 测试要点 / 风险与权衡

## 智能助手
- 文件：`智能助手/智能助手-current.md` · [coding](智能助手/智能助手-coding.md)
- 摘要：个人秘书智能体设计，包含日程管理、待办管理、笔记搜索三大工具能力
- 大纲：背景与目标 / 功能范围 / Agent 执行流程 / 工具定义 / 接口设计 / 类设计 / 数据库设计 / 核心业务规则

## 文档查看器
- 文件：`文档查看器/文档查看器-current.md` · [coding](文档查看器/文档查看器-coding.md)
- 摘要：文档解析与结构化智能体，支持 Markdown 文档解析、目录生成、摘要提取
- 大纲：背景与目标 / 功能范围 / 业务流程 / 接口设计 / 类设计 / 数据库设计 / 核心业务规则

## 大文本块提取工具
- 文件：`大文本块提取工具/大文本块提取工具-current.md` · [coding](大文本块提取工具/大文本块提取工具-coding.md)
- 摘要：基于段落树的大文本智能块提取，支持长文档摘要、结构化分段
- 大纲：背景与目标 / 功能范围 / 算法设计 / 接口设计 / 类设计 / 测试用例

## 通用智能体架构
- 文件：`通用智能体架构/通用智能体架构-current.md`
- 摘要：架构基线文档 — AgentExecutor + ReAct 循环、GraphExecutionEngine DAG 编排、异步执行 + Flux SSE、@Tool 注解驱动注册、@AgentGroup 多智能体编排、三级记忆体系、全链路 Trace、LLMProviderRouter 限速降级
- 大纲：背景 / 功能模块总览图 / Agent ReAct 执行流程 / Graph DAG 编排流程 / 异步执行流程 / 接口设计（6 核心接口）/ 类设计（35+ 类）/ 业务规则（R1-R10）/ Provider 路由降级 / Trace / 记忆体系

### 多平台模型路由层（子模块）
- 文件：`通用智能体架构/多平台模型路由层/多平台模型路由层-current.md` · [coding](通用智能体架构/多平台模型路由层/多平台模型路由层-coding.md)
- 摘要：统一封装 OpenAI/Ollama/阿里云 DashScope 多平台 LLM 调用，支持动态模型切换与成本优化
- 大纲：背景与目标 / 功能范围 / 业务流程 / 接口设计 / 类设计 / 核心业务规则
- 归属原因：模型路由层是通用智能体架构 LLMProvider 环节的基础设施实现

### ChatModel 统一获取与限速降级（子模块）
- 文件：`通用智能体架构/多平台模型路由层/ChatModel统一获取与限速降级-current.md` · [coding](通用智能体架构/多平台模型路由层/ChatModel统一获取与限速降级-coding.md)
- 摘要：消除 AgentExecutor 重复构建 ChatModel 的问题，统一通过 Router 获取；Provider 层增加 Guava RateLimiter 主动限速；Router 层增加 Fallback 降级
- 大纲：背景与目标 / 功能范围 / 业务流程（ChatModel 获取/限速/Fallback）/ 类设计 / 核心业务规则 / 异常处理 / 测试要点
- 归属原因：多平台模型路由层的能力增强，解决 Agent 场景 429 限流问题

### 技术选型（子模块）
- 文件：`通用智能体架构/技术选型/Java大模型应用组件全景图-current.md`
- 摘要：Java 大模型应用 11 层全景图（v4），覆盖接入层到基础设施层全链路候选技术与优缺点 + 多组对比分析表格
- 大纲：全景图 Mermaid / 图例说明 / 框架对比 / 模型 / 向量库 / Embedding / 记忆存储 / 可观测性

- 文件：`通用智能体架构/技术选型/组件复用与框架选型分析.md`
- 摘要：从企业级 Agent 架构视角完整覆盖两大核心框架对比及周边 8 类组件选型决策
- 大纲：全景图速查 / 核心框架对比(Spring AI vs LangChain4j) / 向量数据库 / Embedding模型 / 文档处理 / MCP协议 / 评估与质量 / 安全与合规 / 可观测性 / 全组件决策总表

### 大模型应用开发学习路线
- 文件：`通用智能体架构/大模型应用开发学习路线.md`
- 摘要：从零开始的大模型应用开发学习路径与实战项目推荐
- 大纲：基础准备 / 框架学习 / 实战项目 / 进阶方向

### Agent 全链路 Trace 设计（子模块）
- 文件：`通用智能体架构/Agent全链路Trace设计/Agent全链路Trace设计-current.md`
- 摘要：平台级 Agent 全链路追踪标准 — SpanContext 数据模型 + AgentTraceRecorder 组件（ThreadLocal Span 栈 + SLF4J 日志），一期内存实现，预留 OpenTelemetry 演进路径
- 大纲：背景与目标 / 核心概念（Trace/Span/调用树）/ SpanContext record / AgentTraceRecorder 组件 / 使用模式 / 类清单 / 业务规则 / 演进规划
- 归属原因：从 devplan Trace 设计中提取通用化，所有 Agent Group 共用的可观测性基础设施

### 平台管理接口补全（子模块）
- 文件：`通用智能体架构/平台管理接口补全/平台管理接口补全-current.md` · [coding](通用智能体架构/平台管理接口补全/平台管理接口补全-coding.md)
- 摘要：补全 5 大管理对象的关联查询接口 — Graph 下 Agent 列表、调用链视图、Agent 下 Tool 详情、模型平台清单
- 大纲：背景与目标 / 功能范围 / 4 个新增 GET 接口设计 / 类设计 / 核心业务规则
- 归属原因：管理接口操作的对象（Graph/Agent/Tool/ModelConfig）全部定义在通用智能体架构中

### 角色管理页面（子模块）
- 文件：`通用智能体架构/角色管理页面/角色管理页面-current.md`
- 摘要：前端新增 `/agent-management` 页面 — 仿 TemplateManagement 双栏（左列表 + 右编辑卡片），对接既有 AgentController CRUD，主视觉是 systemPrompt 大 textarea，可多选 toolIds、配置 llmProvider/llmModel/maxIterations/timeoutSeconds；后端零变更
- 大纲：代码入口 / 既有接口契约 / 核心流程图 / 8 条核心规则 / 失败行为 / 视觉参考
- 归属原因：Agent 定义在通用智能体架构中，本页面是其配置入口

### Graph 可视化编排（子模块）
- 文件：`通用智能体架构/Graph可视化编排/Graph可视化编排-current.md` · [coding](通用智能体架构/Graph可视化编排/Graph可视化编排-coding.md)
- 摘要：前端使用 Vue Flow + Dagre 以交互式 DAG 流程图展示 Graph 中 Agent 调度关系和 Agent-Tool 绑定关系
- 大纲：背景与目标 / 功能范围（总览图+能力分解图）/ 业务流程 / 接口设计 / 类设计 / 核心业务规则 / 下游依赖 / 测试要点
- 归属原因：可视化的对象（Graph/Agent/Tool）定义在通用智能体架构中

### Agent 异步执行机制（子模块）
- 文件：`通用智能体架构/Agent异步执行机制/Agent异步执行机制-current.md` · [coding](通用智能体架构/Agent异步执行机制/Agent异步执行机制-coding.md)
- 摘要：Agent 执行从同步阻塞改为异步提交 + 状态轮询 / SSE 推送（Flux 模式）。SSE 分层：AgentSseManager → AgentEventSink + Flux，API 层不再跨层引用 Infrastructure
- 大纲：背景与目标 / 功能范围（5 能力域）/ 业务流程（异步提交+Flux SSE 推送+异常+状态流转）/ 接口设计（3 端点）/ 类设计（13 类）/ 核心业务规则（R1-R9）/ 并发与超时控制 / 异常处理 / 测试要点
- 归属原因：改造的 AgentExecutor、AgentExecutionUseCase、AgentController 均定义在通用智能体架构中

## 代码感知智能开发方案智能体
- 文件：`代码感知智能开发方案智能体/整体方案设计-current.md` · [coding](代码感知智能开发方案智能体/整体方案设计-coding.md)
- 摘要：基于组件全景图 v4 架构（L4 可控多 Agent 系统），引入控制平面 + StateGraph 流程编排 + 4 角色 Agent 协作 + Tool 标准协议 + 三级记忆体系 + 全链路 Trace
- 大纲：背景与目标 / 四角色协作模型 / StateGraph 流程设计 / 接口设计 / 类设计 / 数据库设计 / 核心业务规则 / 全景图映射 / 架构成熟度对标 / Prompt 模板

### Tool 层实现设计（子模块）
- 文件：`代码感知智能开发方案智能体/Tool层与Agent初始化器实现/Tool层实现设计-current.md`
- 摘要：8 个 @Tool 实现（4 元数据提取器 + 1 向量索引器 + 2 检索/读取器 + 1 模板器）+ DevPlanToolRegistry 角色路由 + Schema 校验
- 大纲：背景 / Tool 职责分离原则 / 8 Tool 详细设计 / DevPlanToolRegistry / 类清单 / 技术方案 / 异常处理
- 归属原因：v2 总体架构中 Tool 标准协议层的实现设计

### Agent 初始化器实现设计（子模块）
- 文件：`代码感知智能开发方案智能体/Tool层与Agent初始化器实现/Agent初始化器实现设计-current.md`
- 摘要：DevPlanAgentInitializer 启动时将 4 个角色 Agent 写入 agent_definition 表，幂等设计，启动顺序依赖
- 大纲：背景 / Agent 记录定义 / 启动顺序依赖 / 类清单 / 业务规则 / 异常处理
- 归属原因：v2 总体架构中 Agent 启动注册的实现设计

### 代码感知智能体实现（子模块）
- 文件：`代码感知智能开发方案智能体/代码感知智能体实现/代码感知智能体实现-current.md` · [coding](代码感知智能开发方案智能体/代码感知智能体实现/代码感知智能体实现-coding.md)
- 摘要：ClaudeCodeProfileGenerator 改造 — SDK 桥接模式（主路径）+ CLI 备选、空闲超时替代固定总超时、结构化事件流日志、双模式配置切换
- 大纲：问题分析 / 双模式架构设计 / SDK 桥接脚本协议 / 空闲超时机制 / 配置扩展 / 类设计 / 业务规则 / 异常处理 / 测试要点
- 归属原因：StateGraph 第一站 ScanNode 的 Agent 实现，输出供后续所有 Agent 消费

### 需求分析智能体实现（子模块）
- 文件：`代码感知智能开发方案智能体/需求分析智能体实现/需求分析智能体实现-current.md` · [coding](代码感知智能开发方案智能体/需求分析智能体实现/需求分析智能体实现-coding.md)
- 摘要：适配画像 3 文件拆分，执行模式从"搜索猜测"改为"文档查询 + 搜索验证"，ImpactAnalysis 输出增强（constraintImpacts / crossServiceImpacts / eventImpacts），System Prompt 重构
- 大纲：问题分析 / 文档查询模式 / State 输入适配 / Prompt 重构 / ImpactAnalysis 扩展 / 降级兼容 / 异常处理 / 测试要点
- 归属原因：StateGraph 第二站 AnalyzeNode 的 Agent 实现

### 方案生成智能体实现（子模块）
- 文件：`代码感知智能开发方案智能体/方案生成智能体实现/方案生成智能体实现-current.md`
- 摘要：SOLUTION_ARCHITECT Agent 完整设计 — CodeSearchTool + TemplateRenderTool 编排 + 设计文档 Markdown 输出 + 修正循环机制
- 大纲：技术选型决策 / 角色定位 / 执行流程 / 工具集 / 模板系统 / System Prompt / 修正循环 / State 交互 / Agent 注册 / 类清单
- 归属原因：StateGraph 第三站 DesignNode 的 Agent 实现

### 方案审查智能体实现（子模块）
- 文件：`代码感知智能开发方案智能体/方案审查智能体实现/方案审查智能体实现-current.md`
- 摘要：PLAN_REVIEWER Agent 完整设计 — 无外部 Tool，使用 PlanSensor 传感器链（计算型 + 推理型 + 聚合型）进行方案评审
- 大纲：技术选型决策 / Tool vs Sensor 对比 / 角色定位 / Sensor Chain 设计 / System Prompt / State 交互 / Agent 注册 / 类清单
- 归属原因：StateGraph 第四站 ReviewNode 的 Agent 实现

## LLM 任务管理（可插拔模块）
- 文件：`llm-task-management/llm-task-management-current.md` · [api](llm-task-management/llm-task-management-api-current.md) · [coding](llm-task-management/llm-task-management-coding.md)
- 摘要：**可插拔的独立 LLM 异步任务管理组件**（v2）— 4 个 Maven 模块（core / storage-jpa / spring-boot-starter / web）+ 6 个 SPI 扩展点（Executor / Interceptor / IdGenerator / CancellationStrategy / RetryPolicy / AuditLogger）+ Spring Boot 自动装配 + 默认实现可全部 `@ConditionalOnMissingBean` 替换。底层：DB write-through + Caffeine 热缓存 + Reactor 热流事件双写 + SSE Last-Event-ID 断线重连 + 启动 INTERRUPTED 恢复 + 协作式取消 + 幂等键。业务方接入 5 步零侵入：pom 引入 + `@LlmTaskExecutorType` 注解 + yml 配置 + submit + 前端 SSE
- 大纲：目标边界 / 整体架构（分层视图 + Maven 模块边界视图）/ 9 个模块拆分 / 4 个关键交互时序图（提交执行 / SSE 断线重连 / 重启恢复 / 取消）/ 10 条核心业务规则 / 编码落点（一期内嵌位置 + 未来抽取映射）/ llm_task + llm_task_event 表结构 + 配置 / 风险与待确认 / 验证要点 / **第 10 节可插拔架构（SPI 矩阵 / 默认实现可替换矩阵 / 注解驱动注册 / Spring Boot 自动装配 / 5 步接入 / 跨项目复用）**
- 归属说明：独立顶层主题（非通用智能体架构子模块），因为覆盖范围超出 Agent + 设计目标是跨项目可复用组件。前序 `通用智能体架构/Agent异步执行机制-current.md` 的"后续扩展：执行结果落库 / 队列调度"在此落地并升级为可插拔形态。未来从 llm-orchestration-platform 抽取到独立 Git 仓库 + 内部 Nexus，kai-toolbox / job-interview-log 等可复用

## llm-frontend 重构
- 文件：`llm-frontend重构/llm-frontend重构-current.md`
- 摘要：参考 kai-toolbox/frontend 结构代码（不是 feature 拆分思路），清掉 8 套花哨主题收敛为 light + dark；`src/styles/` 五个 css + themeRegistry → 单一 `src/index.css`（oklch token + .dark）；新增 `src/shell/`(AppShell/Sidebar/TopBar/ThemeToggle)、`src/components/ui/`(shadcn-style button/card/input/badge/separator/sheet)、`src/lib/utils.ts`(cn helper)；`src/views/` 14 个文件名和路由 path 完全保留，仅 Phase 2 替换内部类名
- 大纲：目标与边界 / 整体架构 / 模块拆分 / 关键交互（启动+路由+移动抽屉）/ 9 条核心规则 / 编码落点（三阶段执行）/ 依赖变更 / 风险待确认

## 架构基线
- 文件：`architecture/architecture-current.md`
- 摘要：项目最初的架构基线文档（2026-03-22 版本）
- 大纲：分层划分 / 模块依赖 / 关键组件
