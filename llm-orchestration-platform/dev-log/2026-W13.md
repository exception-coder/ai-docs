# 开发日志 2026-W13（2026-03-23 ~ 2026-03-29）

## 2026-03-29

### 任务描述
新增个人秘书智能体（Secretary Agent），支持日程管理、待办管理、笔记检索、可插拔工具集、长期记忆、跨会话上下文，以及独立秘书前端页面。

### 创建的文件

**Domain 层** (`llm-domain`)：
- `model/SecretaryMemory.java` — 长期记忆模型（userId, type, content, timestamps）
- `model/SecretarySchedule.java` — 日程模型（title, startTime, endTime, reminder, done）
- `model/SecretaryTodo.java` — 待办模型（title, priority, dueDate, done）
- `repository/SecretaryMemoryRepository.java` — 记忆仓储接口
- `repository/SecretaryScheduleRepository.java` — 日程仓储接口
- `repository/SecretaryTodoRepository.java` — 待办仓储接口

**Infrastructure 层** (`llm-infrastructure`)：
- `entity/secretary/SecretaryMemoryEntity.java` — 记忆 JPA 实体
- `entity/secretary/SecretaryScheduleEntity.java` — 日程 JPA 实体
- `entity/secretary/SecretaryTodoEntity.java` — 待办 JPA 实体
- `repository/secretary/SecretaryMemoryJpaRepository.java` — Spring Data JPA
- `repository/secretary/SecretaryScheduleJpaRepository.java`
- `repository/secretary/SecretaryTodoJpaRepository.java`
- `repository/secretary/SecretaryMemoryRepositoryImpl.java` — 仓储实现
- `repository/secretary/SecretaryScheduleRepositoryImpl.java`
- `repository/secretary/SecretaryTodoRepositoryImpl.java`
- `agent/tool/builtin/ScheduleTool.java` — 日程管理 @Tool Bean
- `agent/tool/builtin/TodoTool.java` — 待办管理 @Tool Bean
- `config/SecretaryConfiguration.java` — 工具插拔配置（@ConditionalOnProperty）
- `config/SecretaryAgentInitializer.java` — 启动时注册 secretary-default Agent

**Application 层** (`llm-application`)：
- `service/SecretaryService.java` — 组装 systemPrompt（含长期记忆）→ 调用 AgentExecutor

**API 层** (`llm-api`)：
- `controller/SecretaryController.java` — REST + SSE 流式端点
  - `POST /api/v1/secretary/chat` — 非流式对话
  - `POST /api/v1/secretary/chat/stream` — SSE 流式对话
  - `GET  /api/v1/secretary/memory` — 获取记忆
  - `POST /api/v1/secretary/memory` — 保存记忆
  - `DELETE /api/v1/secretary/memory` — 清除记忆
  - `GET  /api/v1/secretary/tools` — 工具列表

**前端** (`llm-frontend`)：
- `views/Secretary.vue` — 秘书页面（左侧工具+记忆面板，右侧对话区）
- 修改 `router/index.js` — 新增 `/secretary` 路由
- 修改 `App.vue` — 侧边栏新增「个人秘书」菜单
- 修改 `api/index.js` — 新增 `secretaryAPI`

**配置**：
- `llm-starter/src/main/resources/application.yml` — 新增 `secretary.tools.*` 插拔开关
- `llm-api/pom.xml` — 新增 `spring-boot-starter-webflux`（SSE 流式支持）
- `llm-infrastructure/pom.xml` — 新增 `mapstruct` 依赖
- `pom.xml`（根）— 新增 `mapstruct-processor`、`lombok-mapstruct-binding` 到 dependencyManagement，新增 maven-compiler-plugin 配置

**数据库**：
- `docs/init-database.sql` — 新增 `secretary_memory`、`secretary_schedule`、`secretary_todo` 三张表

### 修改的文件
- `llm-infrastructure/src/main/java/.../agent/tool/ToolScanner.java` — 扫描范围从 `@Component` Bean 扩展到所有 Bean，使 @Bean 方法创建的 Secretary 工具也能被扫描注册

### 关键设计决策
- 秘书本质是预配置 Agent（`secretary-default`），复用现有 ReAct 循环和 ToolRegistry 机制，不引入新执行引擎
- 工具通过 `@ConditionalOnProperty` 插拔，配置在 `application.yml`，前端可动态展示启用状态
- `ToolScanner` 改为扫描所有 Bean 而不只是 `@Component`，解决 `@Bean` 方法创建的秘书工具无法被注册的问题
- SSE 流式复用 Chat.vue 的 `fetch` + `ReadableStream` 模式，Controller 层用 `spring-boot-starter-webflux` 的 `Flux` 实现
- 长期记忆通过 `SecretaryMemoryRepository` 持久化到 DB，每次对话前注入 systemPrompt

---

---

## 2026-03-23

### 任务描述
Spring AI Alibaba 全面替换 + 完整 Agent 运行时体系：将原有 Spring AI 1.0.0-M4（OpenAI/Ollama starter + LangChain4j）**完全替换**为 Spring AI Alibaba，并从零构建完整的 Agent 运行时体系（工具注册 + ReAct 循环 + Graph 流程引擎）。

### 创建的文件

**Domain 层** (`llm-domain`)：
- `model/AgentDefinition.java` — Agent 定义模型
- `model/ToolDefinition.java`、`model/ToolType.java` — 工具定义模型和类型枚举
- `model/GraphDefinition.java`、`model/GraphNode.java`、`model/GraphEdge.java`、`model/NodeType.java` — Graph 流程模型
- `model/AgentExecutionResult.java`、`model/ToolCall.java`、`model/GraphExecutionResult.java` — 执行结果模型
- `executor/AgentExecutor.java`、`executor/GraphExecutor.java` — 执行器接口
- `registry/ToolRegistry.java` — 工具注册中心接口
- `repository/AgentDefinitionRepository.java`、`repository/GraphDefinitionRepository.java` — 仓储接口

**Infrastructure 层** (`llm-infrastructure`)：
- `provider/QwenProvider.java` — 通义千问 Provider（基于 Spring AI Alibaba，替换旧 OpenAI/DeepSeek Provider）
- `provider/OllamaProvider.java` — 重写为 Spring AI Alibaba 版（移除 LangChain4j 依赖）
- `agent/tool/Tool.java`、`agent/tool/ToolParam.java` — 工具注解
- `agent/tool/ToolRegistryImpl.java` — 工具注册中心实现（ConcurrentHashMap）
- `agent/tool/ToolExecutor.java` — 工具执行器（反射调用）
- `agent/tool/ToolScanner.java` — 启动时自动扫描 @Tool Bean 注册到 Registry
- `agent/tool/builtin/CalculatorTool.java` — 内置计算器工具
- `agent/tool/builtin/NoteSearchTool.java` — 内置碎片记录搜索工具
- `agent/executor/AlibabaAgentExecutor.java` — Agent 执行引擎（ReAct 循环）
- `agent/graph/GraphExecutionEngine.java` — Graph DAG 执行引擎（顺序/条件/并行）
- `agent/graph/GraphExecutorImpl.java` — GraphExecutor 实现
- `entity/agent/AgentDefinitionEntity.java`、`entity/agent/GraphDefinitionEntity.java` — JPA 实体
- `repository/AgentDefinitionJpaRepository.java`、`repository/GraphDefinitionJpaRepository.java` — JPA Repository
- `repository/AgentDefinitionRepositoryImpl.java`、`repository/GraphDefinitionRepositoryImpl.java` — 仓储实现
- `config/AgentConfiguration.java` — Agent 运行时 Spring 配置

**Application 层** (`llm-application`)：
- `usecase/AgentExecutionUseCase.java` — Agent 执行用例（CRUD + 执行）
- `usecase/GraphOrchestrationUseCase.java` — Graph 编排用例（CRUD + 执行）

**API 层** (`llm-api`)：
- `controller/AgentController.java` — Agent REST 控制器
- `controller/GraphController.java` — Graph REST 控制器
- `controller/ToolController.java` — 工具 REST 控制器

### 修改的文件
- `pom.xml`（根）— 移除 LangChain4j BOM，引入 `spring-ai-alibaba-bom:1.0.0.0`，新增 Spring Snapshots 仓库
- `llm-infrastructure/pom.xml` — 移除旧 Spring AI starter 和 LangChain4j 依赖，引入 `spring-ai-alibaba-starter`
- `config/{dev,test,prod}/spring-ai.yml` — 全部重写为阿里云百炼配置（`DASHSCOPE_API_KEY` 环境变量）
- `application-llm.yml` — 默认 Provider 改为 `alibaba`，默认模型改为 `qwen-plus`
- `application.yml` — required-properties 改为 `spring.ai.alibaba.api-key`
- `config/LLMConfiguration.java` — 移除 OpenAI/DeepSeek 配置，新增 `AlibabaConfig`
- `docs/init-database.sql` — 新增 `agent_definition` 和 `graph_definition` 表 DDL

### 删除的文件
- `provider/OpenAIProvider.java`、`provider/DeepSeekProvider.java`

### 核心设计决策

1. **完全替换旧 AI 依赖**：Spring AI OpenAI starter + LangChain4j 全部移除，统一走 Spring AI Alibaba（DashScope）
2. **ReAct 循环**：`AlibabaAgentExecutor` 实现标准 Thought → Action(ToolCall) → Observation 推理循环，最大迭代次数可配置
3. **@Tool 注解驱动**：工具以 Spring Bean + `@Tool` 方法声明，启动时自动扫描注册，LLM 通过 JSON 格式调用
4. **Graph DAG 引擎**：支持 LLM 节点、TOOL 节点、CONDITION 条件分支节点、PARALLEL 并行节点，节点间通过 `${varName}` 引用上下文变量
5. **DDD 严格分层**：`LLMProvider` 接口和 `LLMOrchestrationService` 无需修改，新 Provider 自动注册

### API 端点（新增）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /api/v1/agents | 获取 Agent 列表 |
| POST | /api/v1/agents | 创建/更新 Agent 定义 |
| GET | /api/v1/agents/{id} | 获取单个 Agent |
| DELETE | /api/v1/agents/{id} | 删除 Agent |
| POST | /api/v1/agents/{id}/execute | 执行 Agent（ReAct 循环） |
| GET | /api/v1/graphs | 获取 Graph 列表 |
| POST | /api/v1/graphs | 创建/更新 Graph 定义 |
| DELETE | /api/v1/graphs/{id} | 删除 Graph |
| POST | /api/v1/graphs/{id}/execute | 执行 Graph（DAG 编排） |
| GET | /api/v1/tools | 获取已注册工具列表 |
| DELETE | /api/v1/tools/{id} | 注销工具 |

### 变更原因

原有架构停留在单次 LLM 调用层面，无真正的 Agent 能力。通过引入 Spring AI Alibaba 替换旧依赖（统一走阿里云百炼），并构建 ReAct Agent 运行时 + Graph 流程编排引擎，将平台升级为真正的 LLM 编排平台，具备工具调用、多步推理、流程编排能力。

---

---

---

## 2026-03-23

### 任务描述
新增智能碎片记录模块，支持随口输入文字或语音，AI 自动分类整理后存储，敏感信息客户端加密

### 创建的文件

**Domain 层**：
- `llm-domain/.../model/Note.java` — 记录领域模型
- `llm-domain/.../model/NoteCategory.java` — 类目领域模型
- `llm-domain/.../model/NoteClassificationResult.java` — AI 分类结果模型
- `llm-domain/.../repository/NoteRepository.java` — 记录仓储接口
- `llm-domain/.../repository/NoteCategoryRepository.java` — 类目仓储接口
- `llm-domain/.../service/NoteClassifier.java` — 分类服务接口

**Infrastructure 层**：
- `llm-infrastructure/.../entity/NoteEntity.java` — 记录 JPA 实体
- `llm-infrastructure/.../entity/NoteCategoryEntity.java` — 类目 JPA 实体
- `llm-infrastructure/.../repository/NoteJpaRepository.java` — 记录 JPA 仓储
- `llm-infrastructure/.../repository/NoteCategoryJpaRepository.java` — 类目 JPA 仓储
- `llm-infrastructure/.../repository/NoteRepositoryImpl.java` — 记录仓储实现
- `llm-infrastructure/.../repository/NoteCategoryRepositoryImpl.java` — 类目仓储实现
- `llm-infrastructure/.../note/NoteClassifierService.java` — 调用 LLM 进行内容分类

**Application 层**：
- `llm-application/.../usecase/NoteUseCase.java` — 记录用例（捕获/查询/删除）
- `llm-application/.../usecase/NoteCategoryUseCase.java` — 类目用例

**API 层**：
- `llm-api/.../controller/NoteController.java` — 记录 REST 控制器
- `llm-api/.../controller/NoteCategoryController.java` — 类目 REST 控制器
- `llm-api/.../dto/NoteInputDTO.java` — 记录输入 DTO
- `llm-api/.../dto/NoteEncryptedInputDTO.java` — 加密记录输入 DTO
- `llm-api/.../dto/NoteResponseDTO.java` — 记录响应 DTO
- `llm-api/.../dto/NoteCategoryDTO.java` — 类目响应 DTO

**前端**：
- `llm-frontend/src/views/NoteCapture.vue` — 碎片记录主页面
- `llm-frontend/src/utils/crypto.js` — AES-GCM 客户端加密工具

**配置文件**：
- `docs/init-database.sql` — 新增 note_category 和 note 表

### 修改的文件
- `llm-application/pom.xml` — 添加 infrastructure 依赖
- `llm-frontend/src/api/index.js` — 添加 noteAPI
- `llm-frontend/src/router/index.js` — 添加 /note-capture 路由
- `llm-frontend/src/App.vue` — 添加导航菜单项

### 核心设计决策

1. **AI 全自动分类**：调用 LLM 分析用户输入，自动判断类目（新类目自动创建）、生成标题/摘要/标签
2. **客户端加密**：使用 Web Crypto API（SubtleCrypto）实现 AES-GCM 256-bit 加密，密钥派生使用 PBKDF2（100000 次迭代），IV 与密文一起 Base64 存储
3. **语音输入**：使用浏览器原生 Web Speech API，无需后端
4. **DDD 架构**：严格遵循 API → Application → Domain ← Infrastructure 依赖方向，`NoteClassifier` 定义在 Domain 层，由 Infrastructure 实现

### API 端点

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | /api/v1/notes | 捕获记录（AI 分类） |
| POST | /api/v1/notes/encrypted | 存储加密记录 |
| GET | /api/v1/notes | 获取记录列表 |
| GET | /api/v1/notes/{id} | 获取单个记录 |
| GET | /api/v1/notes/search | 搜索记录 |
| DELETE | /api/v1/notes/{id} | 删除记录 |
| GET | /api/v1/notes/categories | 获取类目列表 |
| DELETE | /api/v1/notes/categories/{id} | 删除类目 |

### 使用流程

1. 用户输入文字或语音 → POST /api/v1/notes
2. 后端调用 LLM 分类（返回 category、title、summary、isSensitive 等）
3. 若类目不存在，自动创建
4. 若 isSensitive=true，前端弹出密钥输入 → AES-GCM 加密 → POST /api/v1/notes/encrypted
5. 记录存入 DB，列表展示

### 变更原因

实现个人碎片记录智能体，让用户随口说一句就能自动整理归类记录到对应类目，敏感信息（账号密码）客户端加密存储。

---

---
