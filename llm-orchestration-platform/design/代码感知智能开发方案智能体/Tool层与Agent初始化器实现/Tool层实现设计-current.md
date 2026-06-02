# Tool 层实现设计

> 本文档从「Tool 层、Trace 层与 Agent 初始化器实现设计 v2」中拆分而来，聚焦 **Tool 标准协议层**。
> 父文档：`整体方案设计-20260406-v2.md`
> 关联文档：
> - Agent 初始化器 → `Agent初始化器实现设计-20260412-v1.md`（同目录）
> - Trace 全链路追踪 → `通用智能体架构/Agent全链路Trace设计/Agent全链路Trace设计-20260412-v1.md`
> - 原合并文档 → `Tool层与Agent初始化器实现-20260407-v2.md`（归档，不再维护）

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|-------------|
| v1 | 2026-04-12 | zhangkai | 从合并文档拆分独立；内容与 v2 合并文档的 Tool 部分一致 |

---

## 1. 背景与目标

### 1.1 背景

v2 整体框架已搭建完成（ScanNode → DevPlanAgentRouterImpl → AlibabaAgentExecutor 调用链），但 **devplan/tool/ 目录为空** — 所有 Tool 类均未实现，调用链断裂：

```
ScanNode.execute()
  → agentRouter.route(CODE_AWARENESS, state)
    → agentExecutor.execute()
      → agent.toolIds() 对应的 Tool → ❌ 不存在！
```

### 1.2 v1 设计的问题（本次重构原因）

v1 设计中 Tool 试图"理解"代码（如 ProjectScanTool 输出 ProjectStructure 对象、ArchTopologyTool 输出架构判断），这混淆了 Tool 和 Agent 的职责边界：

| 概念 | 正确定位 | v1 的错误 |
|------|----------|-----------|
| Tool | 眼睛 — 机械提取原始数据 | 试图同时做提取+理解 |
| Agent | 大脑 — LLM 理解和判断 | 没有充分利用 LLM 的理解能力 |

**类比：** Claude Code 的 `/init` 命令扫描项目后生成 CLAUDE.md。扫描工具（读目录、读文件）是眼睛，LLM 把扫描结果总结成项目描述才是大脑。

### 1.3 目标

1. 实现 8 个 `@Tool` 标注的工具类（4 个元数据提取器 + 1 个向量索引器 + 2 个检索/读取器 + 1 个模板器）
2. 实现 DevPlanToolRegistry，提供角色→工具映射和 Schema 校验
3. **让 ScanNode 的完整调用链跑通**

### 1.4 设计边界

**本文档包含：**
- 8 个 @Tool 实现 + DevPlanToolRegistry

**本文档不包含：**
- Agent 初始化器 → 见 `Agent初始化器实现设计-20260412-v1.md`
- Trace 全链路追踪 → 见 `通用智能体架构/Agent全链路Trace设计/Agent全链路Trace设计-20260412-v1.md`
- VectorStoreRepository 接口泛化（直接使用 Spring AI VectorStore）
- JavaParser AST 深度分析（采用正则+文件结构方案，后续可增强）
- Sensor 传感器链实现（独立子任务）
- 项目仓库管理（GitLab 仓库注册、定期 clone/pull 同步）— 独立前置模块

**关于 projectPath 的设计假设：**
- `projectPath` 始终指向本地已存在的项目目录
- 代码仓库的 clone/pull 同步是前置运维工作，不在方案生成调用链中触发
- 后续由独立的「项目管理模块」负责：注册项目 → 关联 GitLab 仓库 → 定期同步到本地工作目录

---

## 2. 核心设计原则

**Tool = 眼睛（机械提取），Agent = 大脑（LLM 理解）。** Tool 绝不做判断和总结。

> CODE_AWARENESS Agent 如何编排这些 Tool、生成 ProjectProfile 的完整流程，见独立文档：
> `代码感知智能体实现/代码感知智能体实现-20260408-v1.md`

---

## 3. 详细设计

### 3.1 元数据提取器（4 个）+ 向量索引器（1 个）

#### 3.1.1 ProjectScanTool

**职责：** 机械扫描项目目录结构、Maven 模块、文件统计。

**Tool ID：** `devplan_project_scan`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| projectPath | String | 是 | 项目根目录绝对路径 |

**输出 JSON 结构：**

```json
{
  "projectName": "llm-orchestration-platform",
  "buildTool": "maven",
  "modules": [
    {"name": "llm-domain", "path": "llm-domain", "sourceRoot": "llm-domain/src/main/java"},
    {"name": "llm-application", "path": "llm-application", "sourceRoot": "llm-application/src/main/java"},
    {"name": "llm-infrastructure", "path": "llm-infrastructure", "sourceRoot": "llm-infrastructure/src/main/java"},
    {"name": "llm-api", "path": "llm-api", "sourceRoot": "llm-api/src/main/java"}
  ],
  "packageTree": {
    "com.exceptioncoder.llm": {
      "domain": {"devplan": {}, "model": {}, "service": {}},
      "application": {"devplan": {}},
      "infrastructure": {"devplan": {}, "agent": {}, "vector": {}},
      "api": {"controller": {}, "dto": {}}
    }
  },
  "stats": {
    "totalModules": 4,
    "totalJavaFiles": 205,
    "totalLines": 14609,
    "filesByModule": {"llm-domain": 45, "llm-infrastructure": 95, "llm-application": 30, "llm-api": 35}
  }
}
```

**实现逻辑：**
1. 验证 projectPath 存在且为目录
2. 检测构建工具：有 pom.xml → Maven，有 build.gradle → Gradle
3. 解析根 pom.xml，提取 `<modules>` 列表和 `<artifactId>`
4. 对每个模块，扫描 `src/main/java` 下的包结构和 .java 文件
5. 统计文件数和行数
6. **原样返回 JSON，不做任何业务判断**

---

#### 3.1.2 DependencyAnalysisTool

**职责：** 机械解析 pom.xml，提取依赖清单、版本号、父工程信息。

**Tool ID：** `devplan_dependency_analysis`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| projectPath | String | 是 | 项目根目录 |

**输出 JSON 结构：**

```json
{
  "parent": {
    "groupId": "org.springframework.boot",
    "artifactId": "spring-boot-starter-parent",
    "version": "3.2.2"
  },
  "properties": {
    "java.version": "17",
    "spring-ai.version": "1.0.0-M4"
  },
  "dependencies": [
    {"groupId": "com.alibaba.cloud.ai", "artifactId": "spring-ai-alibaba-starter", "version": "1.0.0.4", "scope": "compile"},
    {"groupId": "org.springframework.ai", "artifactId": "spring-ai-qdrant-store-spring-boot-starter", "scope": "compile"},
    {"groupId": "mysql", "artifactId": "mysql-connector-j", "scope": "runtime"},
    {"groupId": "org.springframework.boot", "artifactId": "spring-boot-starter-data-jpa", "scope": "compile"}
  ],
  "moduleDependencies": {
    "llm-api": ["llm-application", "llm-domain"],
    "llm-application": ["llm-domain"],
    "llm-infrastructure": ["llm-domain"]
  }
}
```

**实现逻辑：**
1. 解析根 pom.xml：提取 parent、properties、dependencyManagement
2. 解析每个模块的 pom.xml：提取 dependencies（合并版本号）
3. 提取模块间依赖关系（`<dependency>` 中引用同项目其他模块的）
4. **原样返回，不判断"这是什么技术栈"**（这是 Agent 的事）

---

#### 3.1.3 CodeStructureAnalysisTool

**职责：** 机械扫描 Java 源文件，提取注解标记的类、类签名、import 关系。

**Tool ID：** `devplan_code_structure`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| projectPath | String | 是 | 项目根目录 |

**输出 JSON 结构：**

```json
{
  "controllers": [
    {
      "className": "DevPlanController",
      "fullClassName": "com.exceptioncoder.llm.api.controller.DevPlanController",
      "filePath": "llm-api/src/main/java/.../DevPlanController.java",
      "annotations": ["@RestController", "@RequestMapping(\"/v1/dev-plan\")"],
      "endpoints": [
        {"method": "POST", "path": "/generate", "methodName": "generateDevPlan"},
        {"method": "GET", "path": "/task/{taskId}", "methodName": "getTaskStatus"}
      ]
    }
  ],
  "entities": [
    {
      "className": "AgentDefinitionEntity",
      "fullClassName": "com.exceptioncoder.llm.infrastructure.entity.agent.AgentDefinitionEntity",
      "tableName": "agent_definition",
      "fields": [
        {"name": "id", "type": "String", "column": "id"},
        {"name": "name", "type": "String", "column": "name"}
      ]
    }
  ],
  "services": [
    {
      "className": "DevPlanUseCase",
      "fullClassName": "com.exceptioncoder.llm.application.usecase.DevPlanUseCase",
      "annotations": ["@Service"],
      "publicMethods": [
        {"name": "generateDevPlan", "params": ["DevPlanRequest"], "returnType": "DevPlanResponse"}
      ]
    }
  ],
  "repositories": [
    {
      "className": "AgentDefinitionJpaRepository",
      "fullClassName": "com.exceptioncoder.llm.infrastructure.repository.AgentDefinitionJpaRepository",
      "entityType": "AgentDefinitionEntity"
    }
  ],
  "layerDependencies": {
    "api": {"imports": ["application", "domain"]},
    "application": {"imports": ["domain"]},
    "domain": {"imports": []},
    "infrastructure": {"imports": ["domain"]}
  },
  "layerViolations": [
    {"from": "domain.xxx", "to": "infrastructure.yyy", "file": "SomeClass.java"}
  ]
}
```

**实现逻辑：**
1. 扫描所有 .java 文件
2. 对每个文件用正则提取：
   - `package` 声明 → 确定所属模块和层
   - 类声明行（`public class/interface/enum XXX`）
   - 类级注解（`@RestController`、`@Entity`、`@Service`、`@Repository`、`@Component`）
   - `@RequestMapping`/`@GetMapping`/`@PostMapping` → 端点信息
   - `@Table(name=xxx)` → 表名
   - `@Column` → 字段映射
   - `public` 方法签名
   - `import` 语句 → 层间依赖
3. 按类别分组（controllers/entities/services/repositories）
4. 聚合 import 关系为层间依赖矩阵
5. 检测违规：domain 包的类 import 了 infrastructure 包
6. **原样返回，"这些类是什么业务能力"由 Agent 判断**

---

#### 3.1.4 ConfigScanTool

**职责：** 机械读取配置文件，提取关键配置项。

**Tool ID：** `devplan_config_scan`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| projectPath | String | 是 | 项目根目录 |

**输出 JSON 结构：**

```json
{
  "profiles": ["default", "dev", "prod"],
  "configFiles": [
    "llm-api/src/main/resources/application.yml",
    "llm-api/src/main/resources/application-dev.yml"
  ],
  "server": {
    "port": 8080,
    "contextPath": "/api"
  },
  "datasource": {
    "url": "jdbc:mysql://localhost:3306/llm_platform",
    "driver": "com.mysql.cj.jdbc.Driver"
  },
  "externalServices": [
    {"name": "qdrant", "config": "spring.ai.vectorstore.qdrant.host=localhost"},
    {"name": "redis", "config": "spring.data.redis.host=localhost"}
  ],
  "customProperties": {
    "llm.alibaba.model": "qwen-max",
    "llm.alibaba.base-url": "https://dashscope.aliyuncs.com/compatible-mode"
  }
}
```

**实现逻辑：**
1. 查找所有 `application*.yml` 和 `application*.properties` 文件
2. 解析 YAML/Properties，提取：
   - `spring.profiles` → Profile 列表
   - `server.*` → 服务器配置
   - `spring.datasource.*` → 数据库配置（脱敏密码）
   - `spring.ai.*` → AI 相关配置
   - `spring.data.redis.*` → Redis 配置
   - 自定义配置前缀
3. **敏感信息脱敏**：password/secret/key 类字段值替换为 `***`
4. **原样返回配置项，不判断"用了什么中间件"**（Agent 的事）

---

#### 3.1.5 CodeIndexTool

**职责：** 将项目 Java 源文件按类级别向量化索引到 Qdrant，供后续 CodeSearchTool 语义检索。

**Tool ID：** `devplan_code_index`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| projectPath | String | 是 | 项目根目录 |
| forceReindex | String | 否 | 是否强制重建索引，默认 "false" |

**输出 JSON：**

```json
{
  "collectionName": "devplan_a1b2c3d4",
  "docCount": 205,
  "status": "READY",
  "skipped": false
}
```

**实现逻辑：**
1. 计算项目文件 hash（所有 .java 文件路径+大小的 MD5）
2. 查询 code_index_status 表，比对 file_hash
   - 一致且非强制 → 返回 `{"status":"READY","skipped":true}`
3. 扫描所有 .java 文件，对每个文件提取：
   - 类级 Javadoc 注释
   - 类声明行（class/interface/enum + extends/implements）
   - public 方法签名列表
4. 拼接 embedding 文本：`"{Javadoc}\n{类声明}\n{方法签名列表}"`
5. 创建 Spring AI Document（metadata: filePath, className, packageName）
6. 批量写入 VectorStore
7. 更新 code_index_status 表

**关键依赖：**
- `org.springframework.ai.vectorstore.VectorStore` — Spring AI Qdrant 自动配置
- `CodeIndexStatusRepository` — JPA

---

### 3.2 检索与读取工具

#### 3.2.1 CodeSearchTool

**职责：** 基于语义在已索引代码中检索相关类。供 REQUIREMENT_ANALYZER 和 SOLUTION_ARCHITECT 共用。

**Tool ID：** `devplan_code_search`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| query | String | 是 | 搜索查询 |
| topK | String | 否 | 结果数量，默认 "5" |

**输出 JSON：** 匹配结果数组（className, filePath, score, snippet）

**实现逻辑：**
1. VectorStore.similaritySearch(query, topK) 检索
2. 从 Document metadata 提取 className, filePath, packageName
3. 组装结果 JSON 返回

**降级策略：** VectorStore 不可用 → 返回空结果 + WARN 日志

---

#### 3.2.2 FileReadTool

**职责：** 读取指定文件内容，供 Agent 查看具体代码。

**Tool ID：** `devplan_file_read`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| filePath | String | 是 | 文件绝对路径 |
| maxLines | String | 否 | 最大行数，默认 "200" |

**安全约束：** 路径必须在项目目录下，防止路径遍历。

---

### 3.3 模板渲染工具

#### 3.3.1 TemplateRenderTool

**职责：** 用预定义模板渲染设计文档骨架。

**Tool ID：** `devplan_template_render`

**输入参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| templateName | String | 是 | 模板名（"STANDARD"/"LIGHTWEIGHT"） |
| context | String | 是 | JSON 格式的模板变量 |

**输出：** 渲染后的 Markdown 文本。

**模板存储：** 内置 Java 常量（后续可迁移到 prompt_template 表）。

---

### 3.4 DevPlanToolRegistry

**职责：** Tool 标准协议层 — 按角色路由工具集 + JSON Schema 参数校验。

**类名：** `com.exceptioncoder.llm.infrastructure.devplan.tool.DevPlanToolRegistry`

```java
private static final Map<AgentRole, List<String>> ROLE_TOOL_MAPPING = Map.of(
    CODE_AWARENESS, List.of(
        "devplan_project_scan",
        "devplan_dependency_analysis",
        "devplan_code_structure",
        "devplan_config_scan",
        "devplan_code_index"
    ),
    REQUIREMENT_ANALYZER, List.of(
        "devplan_code_search",
        "devplan_file_read"
    ),
    SOLUTION_ARCHITECT, List.of(
        "devplan_code_search",
        "devplan_template_render"
    ),
    PLAN_REVIEWER, List.of()  // 无外部工具，纯 LLM 评审
);
```

**方法：**
- `getToolsForRole(AgentRole)` → 返回该角色可用的 ToolDefinition 列表
- `validateAndExecute(toolName, params, callerRole)` → 权限检查 + Schema 校验 + 执行

---

## 4. 类清单（全类名）

| 全类名 | 操作 | 说明 |
|--------|------|------|
| **Tool — 元数据提取器 + 向量索引器** | | |
| `c.e.l.infrastructure.devplan.tool.ProjectScanTool` | 新建 | @Tool, 目录结构+模块+文件统计 |
| `c.e.l.infrastructure.devplan.tool.DependencyAnalysisTool` | 新建 | @Tool, pom.xml 解析→依赖清单 |
| `c.e.l.infrastructure.devplan.tool.CodeStructureAnalysisTool` | 新建 | @Tool, 注解扫描+类签名+import 分析+层依赖 |
| `c.e.l.infrastructure.devplan.tool.ConfigScanTool` | 新建 | @Tool, 配置文件提取+脱敏 |
| `c.e.l.infrastructure.devplan.tool.CodeIndexTool` | 新建 | @Tool, 代码向量索引到 Qdrant |
| **Tool — 检索 / 读取 / 渲染** | | |
| `c.e.l.infrastructure.devplan.tool.CodeSearchTool` | 新建 | @Tool, VectorStore 语义搜索 |
| `c.e.l.infrastructure.devplan.tool.FileReadTool` | 新建 | @Tool, 安全文件读取 |
| `c.e.l.infrastructure.devplan.tool.TemplateRenderTool` | 新建 | @Tool, 设计文档模板渲染 |
| **标准协议** | | |
| `c.e.l.infrastructure.devplan.tool.DevPlanToolRegistry` | 新建 | @Component, 角色→工具映射+Schema 校验 |

> 注：`c.e.l` = `com.exceptioncoder.llm`

**复用类：**

| 全类名 | 复用方式 |
|--------|----------|
| `c.e.l.infrastructure.agent.tool.Tool` / `ToolParam` | @Tool/@ToolParam 注解 |
| `c.e.l.infrastructure.agent.tool.ToolScanner` | 启动时自动发现 |
| `c.e.l.infrastructure.agent.tool.ToolRegistryImpl` | 内存注册表 |
| `c.e.l.infrastructure.agent.tool.ToolExecutor` | 反射执行 |
| `org.springframework.ai.vectorstore.VectorStore` | Spring AI Qdrant |
| `c.e.l.infrastructure.devplan.repository.CodeIndexStatusRepository` | 索引状态 JPA |

---

## 5. 技术方案

### 5.1 关键技术决策

| 决策点 | 方案 | 原因 |
|--------|------|------|
| 代码向量化 | 直接使用 Spring AI `VectorStore` | 现有 VectorStoreRepository 耦合 JobPosting |
| 代码结构分析 | 正则+文件结构（不引入 JavaParser） | 减少依赖复杂度，正则覆盖 95% 场景 |
| 配置解析 | SnakeYAML（Spring 已内置） | 无新增依赖 |
| pom.xml 解析 | JDK 内置 DOM XML Parser | 无新增依赖 |
| Tool 命名 | `devplan_` 前缀 | 与现有 `schedule_*`/`todo_*` 一致 |

### 5.2 新增依赖

**无新增 Maven 依赖。** 全部基于：
- Spring AI Qdrant（已有）
- JDK NIO（文件操作）
- JDK DOM Parser（XML 解析）
- JDK Regex（代码结构分析）
- SnakeYAML（Spring 内置，YAML 解析）

---

## 6. 核心业务规则

| # | 规则 | 来源 |
|---|------|------|
| R1 | **Tool 只做机械提取，不做理解和判断** | 本文档核心原则 |
| R2 | Tool 命名 `devplan_` 前缀 | 本文档 |
| R3 | 索引粒度：类级别，embedding = Javadoc + 类声明 + public 方法签名 | v2 规则6 |
| R4 | 索引去重：同 projectPath 只维护一个 collection，基于 file_hash 判断 | v2 规则5 |
| R5 | 角色工具隔离：Agent 只能调用其角色对应工具集 | v2 规则3 |
| R6 | Schema 校验：@Tool 调用前通过参数校验 | v2 规则4 |
| R7 | 向量库降级：不可用时返回空结果 + WARN，不阻断 | v2 约束 |
| R8 | FileReadTool 安全：路径必须在项目目录下 | 本文档 |
| R9 | ConfigScanTool 脱敏：password/secret/key → *** | 本文档 |

---

## 7. 异常处理

| 场景 | 处理方式 |
|------|----------|
| projectPath 不存在 | Tool 返回 `{"error": "路径不存在"}` 而非抛异常（Agent 可观察并调整） |
| 非 Maven 项目 | Tool 返回 `{"error": "未检测到 pom.xml"}`（后续扩展 Gradle 支持） |
| pom.xml 解析失败 | DependencyAnalysisTool 返回 `{"error": "pom.xml 解析失败", "detail": "..."}` |
| 无 .java 文件 | 返回空结构 + `{"warning": "未找到 Java 源文件"}` |
| Qdrant 不可用 | CodeIndexTool/CodeSearchTool 降级返回空结果 + WARN 日志 |
| 配置文件不存在 | ConfigScanTool 返回 `{"warning": "未找到配置文件", "configFiles": []}` |
| 文件路径遍历 | FileReadTool 返回 `{"error": "路径安全检查失败"}` |
| 工具越权 | DevPlanToolRegistry 抛 ToolPermissionDeniedException |
| Schema 校验失败 | 返回校验错误信息 |

---

## 8. 对 v2 父文档的变更影响

本文档的 Tool 重新设计与 v2 父文档的类清单存在差异：

| v2 父文档中的类 | 本文档处理 |
|----------------|-----------|
| ProjectScanTool | 保留，职责收窄为纯机械扫描 |
| ArchTopologyTool | **合并**进 CodeStructureAnalysisTool（层依赖分析是 import 扫描的一部分） |
| CodeIndexTool | 保留 |
| CodeSearchTool | 保留 |
| FileReadTool | 保留 |
| TemplateRenderTool | 保留 |
| — | **新增** DependencyAnalysisTool（pom.xml 解析） |
| — | **新增** CodeStructureAnalysisTool（注解扫描+import 分析） |
| — | **新增** ConfigScanTool（配置提取） |

**注意：** v2 父文档中 CODE_AWARENESS Agent 的 SystemPrompt 需配合更新，增加对新工具的说明。此处不修改父文档，待实现验证后统一同步。

---

## 9. 测试要点

| 测试项 | 类型 | 说明 |
|--------|------|------|
| ProjectScanTool 扫描多模块 Maven 项目 | 单元测试 | 临时目录构造测试项目 |
| DependencyAnalysisTool 解析 pom.xml | 单元测试 | 验证依赖提取和版本号合并 |
| CodeStructureAnalysisTool 注解扫描 | 单元测试 | 验证 @Controller/@Entity 识别准确性 |
| CodeStructureAnalysisTool 层违规检测 | 单元测试 | 构造 domain→infra 的 import 违规 |
| ConfigScanTool 敏感信息脱敏 | 单元测试 | 验证 password 字段被替换为 *** |
| CodeIndexTool 索引去重 | 单元测试 | mock VectorStore，验证 file_hash 比对 |
| DevPlanToolRegistry 角色路由 | 单元测试 | 每个角色返回正确工具集 |
| DevPlanToolRegistry 越权拒绝 | 单元测试 | PLAN_REVIEWER 调 ProjectScanTool → 拒绝 |
