# 开发日志 2026-W08（2026-02-18 ~ 2026-02-19）

## 2026-02-19

### 任务描述
新增 DeepSeek Provider 支持，扩展 LLM 编排平台的模型提供商能力

### 新增文件
- `llm-infrastructure/src/main/java/com/enterprise/llm/infrastructure/provider/DeepSeekProvider.java`

### 修改文件
- `llm-infrastructure/src/main/java/com/enterprise/llm/infrastructure/config/LLMConfiguration.java`：添加 DeepSeekConfig 配置类

### 核心功能

1. **DeepSeek API 集成**
   - DeepSeek API 兼容 OpenAI 格式
   - 使用 Spring AI 的 OpenAI 客户端进行调用
   - 支持同步和流式两种调用方式

2. **配置管理**
   - API Key 配置
   - Base URL 配置（默认：https://api.deepseek.com）
   - 默认模型：deepseek-chat
   - 温度参数：0.7
   - 最大 Token 数：4000

3. **消息转换**
   - 支持 system、user、assistant 三种角色
   - 自动转换为 Spring AI 的消息格式

4. **响应处理**
   - 解析 Token 使用情况
   - 提取完成原因
   - 统一返回 LLMResponse 格式

### 核心设计决策

1. **复用 OpenAI 客户端**
   - DeepSeek API 完全兼容 OpenAI 格式
   - 通过配置不同的 Base URL 和 API Key 即可使用
   - 避免重复开发，提高代码复用性

2. **懒加载模式**
   - ChatModel 采用懒加载 + 单例模式
   - 首次调用时创建，后续复用
   - 使用 synchronized 确保线程安全

3. **模型识别**
   - `supports()` 方法检查模型名称是否包含 "deepseek"
   - 自动路由到 DeepSeek Provider

4. **参数优先级**
   - 请求参数 > 配置文件默认值
   - 灵活支持不同场景的参数需求

5. **错误处理**
   - 统一异常捕获和日志记录
   - 流式调用失败返回 Flux.error

### 使用配置示例

```yaml
llm:
  deepseek:
    api-key: sk-xxxxx
    base-url: https://api.deepseek.com
    model: deepseek-chat
    temperature: 0.7
    max-tokens: 4000
```

### 支持的模型
- deepseek-chat
- deepseek-coder
- 其他 DeepSeek 系列模型

### 变更原因
用户需要使用 DeepSeek 模型进行 LLM 调用。DeepSeek 提供高性价比的大语言模型服务，API 兼容 OpenAI 格式，易于集成。通过添加 DeepSeek Provider，扩展平台的模型选择范围，满足不同场景的需求。

---

---

---

## 2026-02-19 (更新)

### 任务描述
重构 Prompt 测试和模板管理功能，移除硬编码，改为从数据库动态获取配置

### 新增文件

1. **基础设施层**
   - `LLMModelConfigEntity.java`：LLM 模型配置实体
   - `LLMModelConfigJpaRepository.java`：模型配置 JPA Repository

2. **应用层**
   - `LLMModelConfigUseCase.java`：模型配置管理用例

3. **API 层**
   - `LLMModelConfigController.java`：模型配置管理 REST 控制器

4. **数据库脚本**
   - `init-model-config.sql`：模型配置表初始化脚本，包含 OpenAI、DeepSeek、Ollama 等模型的初始数据

### 修改文件

1. **PromptTemplateEntity.java**
   - 添加 `variableExamples` 字段，用于存储 JSON 格式的变量示例

2. **DatabasePromptTemplate.java**
   - 添加 `variableExamples` 字段

3. **DatabasePromptTemplateFactory.java**
   - 添加支持 `variableExamples` 参数的工厂方法

4. **PromptTemplateRepositoryImpl.java**
   - 保存和转换时处理 `variableExamples` 字段

5. **PromptTemplateManagementUseCase.java**
   - `TemplateInfo` 和 `TemplateCreateRequest` 添加 `variableExamples` 字段
   - 添加 `getTemplateVariableExamples()` 方法
   - 更新 `saveTemplate()` 方法支持变量示例
   - 更新 `convertToInfo()` 方法处理变量示例

6. **PromptTestController.java**
   - 移除硬编码的模型列表，改为调用 `LLMModelConfigUseCase.getAllEnabledModels()`
   - 移除硬编码的变量示例，改为从模板的 `variableExamples` 字段读取
   - 使用 Jackson ObjectMapper 解析 JSON 格式的变量示例

7. **PromptTemplateController.java**
   - `TemplateRequest` 添加 `variableExamples` 字段
   - 更新 `saveTemplate()` 方法传递变量示例

### 核心设计决策

1. **模型配置数据库化**
   - 创建 `llm_model_config` 表管理可用模型
   - 支持启用/禁用、排序、分类等功能
   - 前端可动态配置，无需修改代码

2. **变量示例与模板绑定**
   - 在 `prompt_template` 表添加 `variable_examples` 字段
   - 使用 JSON 格式存储，灵活支持不同结构
   - 创建模板时统一配置变量示例

3. **移除硬编码**
   - Prompt 测试的模型列表从数据库读取
   - 模板变量示例从数据库读取
   - 提高系统灵活性和可维护性

4. **JSON 格式处理**
   - 使用 Jackson ObjectMapper 解析 JSON
   - 统一错误处理，解析失败时返回友好提示

5. **RESTful API 设计**
   - 新增 `/api/v1/model-config` 端点管理模型配置
   - 支持 CRUD 操作
   - 支持按提供商筛选

### 数据库变更

1. **新增表**：`llm_model_config`
   - 存储模型代码、提供商、名称、描述等信息
   - 支持启用/禁用和排序

2. **修改表**：`prompt_template`
   - 添加 `variable_examples` 字段

3. **初始数据**
   - 预置 OpenAI（GPT-4、GPT-3.5-turbo 等）
   - 预置 DeepSeek（deepseek-chat、deepseek-coder）
   - 预置 Ollama（Llama、Mistral、Qwen 等）

### API 端点

**模型配置管理**：
- `GET /api/v1/model-config` - 获取所有启用的模型
- `GET /api/v1/model-config/provider/{provider}` - 按提供商获取模型
- `GET /api/v1/model-config/{modelCode}` - 获取单个模型
- `POST /api/v1/model-config` - 创建或更新模型
- `DELETE /api/v1/model-config/{modelCode}` - 删除模型

**Prompt 测试**（已更新）：
- `GET /api/v1/prompt-test/models` - 从数据库获取模型列表
- `GET /api/v1/prompt-test/template-variables/{templateName}` - 从数据库获取变量示例

### 变更原因

原有的硬编码方式不够灵活，每次添加新模型或修改变量示例都需要修改代码并重新部署。通过数据库化配置，实现了：

1. **动态配置**：前端可直接管理模型和变量示例
2. **易于维护**：无需修改代码，降低维护成本
3. **更好的扩展性**：支持任意数量的模型和提供商
4. **统一管理**：模板和变量示例绑定，避免不一致

这次重构遵循了 DDD-lite 架构原则，保持了清晰的分层和依赖方向。

---

---

---

## 2026-02-19 (更新2)

### 任务描述
修复编译错误，确保架构分层正确

### 新增文件

1. **领域层**
   - `LLMModelConfig.java`：LLM 模型配置领域模型
   - `LLMModelConfigRepository.java`：LLM 模型配置仓储接口

2. **基础设施层**
   - `LLMModelConfigRepositoryImpl.java`：模型配置仓储实现

### 修改文件

1. **PromptTemplateFactory.java**（领域层）
   - 添加重载方法，支持带 `variableExamples` 参数的模板创建

2. **DatabasePromptTemplateFactory.java**（基础设施层）
   - 实现接口的重载方法，使用 `@Override` 注解

3. **LLMModelConfigUseCase.java**（应用层）
   - 修改依赖：从直接依赖 `LLMModelConfigJpaRepository`（Infrastructure 层）改为依赖 `LLMModelConfigRepository`（Domain 层）
   - 修改实体类型：从 `LLMModelConfigEntity` 改为 `LLMModelConfig` 领域模型

### 核心设计决策

1. **修复架构违规**
   - Application 层不应该直接依赖 Infrastructure 层的实体和 Repository
   - 通过在 Domain 层定义接口和模型，Infrastructure 层实现接口
   - 保持正确的依赖方向：Application → Domain ← Infrastructure

2. **接口方法重载**
   - 在 `PromptTemplateFactory` 接口中添加重载方法
   - 保持向后兼容，原有方法继续可用
   - 新方法支持变量示例参数

3. **实体转换**
   - Infrastructure 层负责 Entity 和 Domain Model 之间的转换
   - Application 层只使用 Domain Model
   - 保持各层职责清晰

### 编译错误修复

1. **LLMModelConfigUseCase 编译错误**
   - 问题：Application 层直接依赖 Infrastructure 层的 `LLMModelConfigEntity` 和 `LLMModelConfigJpaRepository`
   - 解决：创建 Domain 层的 `LLMModelConfig` 模型和 `LLMModelConfigRepository` 接口
   - 结果：依赖方向正确，符合 DDD-lite 架构

2. **PromptTemplateManagementUseCase 编译错误**
   - 问题：调用 `templateFactory.createTemplate()` 时传入了 5 个参数，但接口只定义了 4 个参数的方法
   - 解决：在接口中添加重载方法，支持 5 个参数
   - 结果：方法签名匹配，编译通过

3. **DatabasePromptTemplateFactory 编译错误**
   - 问题：实现类的方法没有使用 `@Override` 注解，且不在接口中
   - 解决：在接口中添加方法定义，实现类使用 `@Override`
   - 结果：符合接口规范

### 架构原则

**正确的依赖方向**：
```
API Layer → Application Layer → Domain Layer
                                      ↑
                          Infrastructure Layer
```

**各层职责**：
- Domain Layer：定义接口、模型、业务规则
- Application Layer：编排用例，依赖 Domain 接口
- Infrastructure Layer：实现 Domain 接口，处理外部系统
- API Layer：处理 HTTP 协议，依赖 Application

### 变更原因
修复编译错误，确保代码符合 DDD-lite 架构原则。Application 层不应该直接依赖 Infrastructure 层的具体实现，而应该通过 Domain 层的接口进行交互。这样可以提高代码的可测试性、可维护性和可扩展性。

---

---

---

## 2026-02-19 (更新3)

### 任务描述
修复 PromptTemplateManagementUseCase 中的架构违规问题

### 修改文件

1. **PromptTemplate.java**（领域层接口）
   - 添加 `getVariableExamples()` 方法定义

2. **PromptTemplateManagementUseCase.java**（应用层）
   - 移除对 `DatabasePromptTemplate` 的直接引用
   - 使用接口方法 `template.getVariableExamples()` 替代类型转换
   - 简化 `convertToInfo()` 方法，直接调用接口方法

### 核心设计决策

1. **接口完整性**
   - Domain 层的接口应该包含所有必要的方法
   - 避免在 Application 层使用 `instanceof` 和类型转换
   - 保持接口的完整性和一致性

2. **架构纯净性**
   - Application 层只依赖 Domain 层的接口
   - 不引用任何 Infrastructure 层的具体实现
   - 通过接口方法获取所有需要的数据

3. **代码简洁性**
   - 移除复杂的类型判断和转换逻辑
   - 代码更简洁、更易维护
   - 提高可测试性

### 修复前后对比

**修复前（错误）**：
```java
// Application 层直接引用 Infrastructure 层的类
if (template instanceof com.exceptioncoder.llm.infrastructure.prompt.DatabasePromptTemplate) {
    variableExamples = ((DatabasePromptTemplate) template).getVariableExamples();
}
```

**修复后（正确）**：
```java
// Application 层只使用 Domain 层的接口
String variableExamples = template.getVariableExamples();
```

### 架构原则总结

**严格的分层依赖**：
- Domain Layer：定义完整的接口契约
- Application Layer：只依赖 Domain 接口，不依赖 Infrastructure
- Infrastructure Layer：实现 Domain 接口的所有方法

**禁止的依赖**：
- ❌ Application → Infrastructure
- ❌ Domain → Infrastructure
- ❌ Domain → Application

**允许的依赖**：
- ✅ API → Application
- ✅ API → Domain
- ✅ Application → Domain
- ✅ Infrastructure → Domain

### 变更原因
修复 Application 层直接引用 Infrastructure 层具体实现类的架构违规问题。通过在 Domain 层接口中添加 `getVariableExamples()` 方法，确保 Application 层只依赖接口，保持架构的纯净性和可维护性。

---

---

---

## 2026-02-19 (更新4)

### 任务描述
前端新增模型管理模块，并更新模板管理支持变量示例配置

### 新增文件

1. **前端页面**
   - `ModelManagement.vue`：模型管理页面，支持模型的 CRUD 操作

### 修改文件

1. **前端路由**
   - `router/index.js`：添加模型管理路由

2. **前端 API**
   - `api/index.js`：添加 `modelConfigAPI`，包含模型管理的所有 API 方法

3. **前端布局**
   - `App.vue`：添加模型管理菜单项

4. **模板管理页面**
   - `TemplateManagement.vue`：
     - 添加变量示例输入框
     - 添加 JSON 格式化功能
     - 保存时验证 JSON 格式
     - 调整模板内容和变量示例的行数分配

### 核心功能

**模型管理模块**：
1. **模型列表展示**
   - 支持按提供商筛选（全部、OpenAI、DeepSeek、Ollama）
   - 显示模型名称、提供商、启用状态
   - 高亮当前选中的模型

2. **模型 CRUD 操作**
   - 创建新模型：输入模型代码、提供商、名称、描述等
   - 编辑模型：修改模型信息（模型代码不可修改）
   - 删除模型：确认后删除
   - 启用/禁用模型：通过开关控制

3. **排序管理**
   - 支持设置排序顺序（0-1000）
   - 数字越小越靠前

4. **提供商标签**
   - OpenAI：绿色标签
   - DeepSeek：橙色标签
   - Ollama：灰色标签

**模板管理增强**：
1. **变量示例配置**
   - 新增变量示例输入框（JSON 格式）
   - 用于 Prompt 测试时的默认变量值
   - 保存时验证 JSON 格式

2. **JSON 格式化**
   - 一键格式化 JSON
   - 自动缩进和美化
   - 格式错误时提示

3. **表单验证**
   - 模板名称必填
   - 模板内容必填
   - 变量示例必须是有效 JSON

### API 端点

**模型管理**：
- `GET /api/v1/model-config` - 获取所有模型
- `GET /api/v1/model-config/provider/{provider}` - 按提供商获取
- `GET /api/v1/model-config/{modelCode}` - 获取单个模型
- `POST /api/v1/model-config` - 创建或更新模型
- `DELETE /api/v1/model-config/{modelCode}` - 删除模型

**模板管理**（已更新）：
- `POST /api/v1/prompt-templates` - 保存模板（支持 variableExamples 字段）

### 用户体验优化

1. **友好的交互**
   - 创建/编辑/查看模式切换流畅
   - 操作成功/失败有明确提示
   - 删除操作需要二次确认

2. **数据验证**
   - 前端验证必填字段
   - JSON 格式实时验证
   - 错误提示清晰明确

3. **视觉设计**
   - 使用标签区分不同提供商
   - 启用/禁用状态一目了然
   - 帮助文本引导用户输入

### 使用场景

**模型管理**：
1. 添加新的 LLM 模型（如新版本的 GPT）
2. 启用/禁用某些模型
3. 调整模型在列表中的显示顺序
4. 管理不同提供商的模型

**模板管理**：
1. 创建模板时配置变量示例
2. 在 Prompt 测试时自动填充默认值
3. 方便团队成员理解模板的使用方式

### 变更原因

1. **模型管理需求**：用户需要动态管理可用的 LLM 模型，而不是硬编码在代码中。通过前端界面可以方便地添加、修改、删除模型配置。

2. **变量示例需求**：模板的变量示例可以帮助用户快速理解模板的使用方式，在 Prompt 测试时提供默认值，提升用户体验。

3. **统一管理**：将模型配置和模板配置都通过前端界面管理，降低维护成本，提高系统的灵活性。

---

---

---

## 2026-02-19 (更新5)

### 任务描述
修复 Repository 层 save 方法的严重逻辑错误

### 问题描述

在 `PromptTemplateRepositoryImpl` 和 `LLMModelConfigRepositoryImpl` 的 `save()` 方法中存在严重的逻辑错误：

1. **没有区分新增和更新**：无论记录是否存在，都直接创建新实体并调用 `save()`
2. **可能导致主键冲突**：如果数据库中已存在相同的唯一键（如 `templateName` 或 `modelCode`），会抛出异常
3. **版本号不递增**：更新时版本号始终为 1，无法追踪变更历史
4. **缓存失效但逻辑错误**：虽然使用了 `@CacheEvict` 清空缓存，但底层保存逻辑有问题

### 修复内容

**PromptTemplateRepositoryImpl.save()**：
```java
// 修复前：直接创建新实体
PromptTemplateEntity entity = PromptTemplateEntity.builder()
    .templateName(...)
    .version(1)  // 始终为 1
    .build();

// 修复后：判断是否存在
Optional<PromptTemplateEntity> existingOpt = jpaRepository.findByTemplateNameAndEnabledTrue(...);
if (existingOpt.isPresent()) {
    // 更新现有记录
    entity = existingOpt.get();
    entity.setTemplateContent(...);
    entity.setVersion(entity.getVersion() + 1);  // 版本号递增
} else {
    // 创建新记录
    entity = PromptTemplateEntity.builder()...build();
}
```

**LLMModelConfigRepositoryImpl.save()**：
```java
// 修复前：直接创建新实体
LLMModelConfigEntity entity = LLMModelConfigEntity.builder()
    .modelCode(...)
    .build();

// 修复后：判断是否存在
Optional<LLMModelConfigEntity> existingOpt = jpaRepository.findByModelCode(...);
if (existingOpt.isPresent()) {
    // 更新现有记录
    entity = existingOpt.get();
    entity.setProvider(...);
    entity.setModelName(...);
} else {
    // 创建新记录
    entity = LLMModelConfigEntity.builder()...build();
}
```

### 修复后的逻辑

1. **查询现有记录**：先根据唯一键查询是否已存在
2. **区分新增和更新**：
   - 如果存在：获取现有实体，更新字段值，保留 ID 和审计字段
   - 如果不存在：创建新实体，设置初始值
3. **版本号管理**：更新时版本号递增，便于追踪变更历史
4. **保留审计信息**：更新时保留 `createdAt`、`createdBy` 等字段

### 核心改进

1. **避免主键冲突**：不会因为重复的唯一键导致数据库异常
2. **正确的更新语义**：更新操作真正更新现有记录，而不是尝试插入
3. **版本追踪**：每次更新版本号递增，便于审计和回滚
4. **日志清晰**：明确记录是"创建新记录"还是"更新现有记录"

### 潜在问题分析

**为什么之前的代码可能"看起来能工作"**：
- JPA 的 `save()` 方法会检查实体是否有 ID
- 如果没有 ID，执行 INSERT
- 如果有 ID，执行 UPDATE
- 但我们的代码总是创建新实体（没有 ID），所以总是尝试 INSERT
- 如果唯一键冲突，会抛出 `DataIntegrityViolationException`

**修复后的优势**：
- 明确的业务语义：新增就是新增，更新就是更新
- 避免数据库异常：不会因为唯一键冲突导致失败
- 更好的可维护性：代码逻辑清晰，易于理解和调试

### 变更原因

这是一个严重的逻辑错误，会导致：
1. 更新操作失败（唯一键冲突）
2. 版本号无法正确追踪
3. 审计信息丢失

通过修复，确保 Repository 层的 save 方法能够正确处理新增和更新两种场景，符合业务预期。

---

---

---

## 2026-02-19 (更新6)

### 任务描述
优化对话页面，添加 Markdown 渲染和流式输出支持

### 新增依赖

**前端依赖**：
- `marked@^11.1.1`：Markdown 解析和渲染
- `highlight.js@^11.9.0`：代码高亮

### 修改文件

**前端**：
1. **package.json**
   - 添加 marked 和 highlight.js 依赖

2. **Chat.vue**
   - 添加 Markdown 渲染功能
   - 改为流式输出模式
   - 使用 Fetch API 读取 SSE 流
   - 添加完整的 Markdown 样式（标题、列表、代码块、表格等）
   - 实时追加流式内容并滚动到底部

**后端**：
1. **ChatController.java**
   - 改为 SSE（Server-Sent Events）流式输出
   - 支持 stream 参数控制流式/非流式
   - 使用 SseEmitter 发送数据
   - 异步处理流式响应

2. **ChatRequest.java**
   - 添加 `stream` 字段

3. **ChatUseCase.java**
   - 添加 `executeStream()` 方法
   - 流式完成后保存完整对话历史
   - 使用 AtomicReference 累积流式内容

### 核心功能

**Markdown 渲染**：
1. **支持的语法**
   - 标题（H1-H6）
   - 段落和换行
   - 列表（有序、无序）
   - 代码块（带语法高亮）
   - 行内代码
   - 引用块
   - 表格
   - 链接

2. **代码高亮**
   - 使用 highlight.js
   - 支持多种编程语言
   - GitHub Dark 主题

3. **样式优化**
   - 用户消息和 AI 消息分别适配
   - 代码块深色背景
   - 表格边框和样式
   - 链接颜色和悬停效果

**流式输出**：
1. **前端实现**
   - 使用 Fetch API 读取流
   - 解析 SSE 格式数据
   - 实时追加内容到消息
   - 自动滚动到底部

2. **后端实现**
   - 使用 SseEmitter 发送事件
   - 异步处理避免阻塞
   - 发送 `[DONE]` 标记表示完成
   - 支持错误处理和超时

3. **数据格式**
   ```
   data: {"content": "文本片段"}
   
   data: {"content": "更多文本"}
   
   data: [DONE]
   ```

### 技术实现

**Markdown 配置**：
```javascript
marked.setOptions({
  highlight: function(code, lang) {
    if (lang && hljs.getLanguage(lang)) {
      return hljs.highlight(code, { language: lang }).value
    }
    return hljs.highlightAuto(code).value
  },
  breaks: true,  // 支持 GFM 换行
  gfm: true      // GitHub Flavored Markdown
})
```

**流式读取**：
```javascript
const reader = response.body.getReader()
const decoder = new TextDecoder()

while (true) {
  const { done, value } = await reader.read()
  if (done) break
  
  const chunk = decoder.decode(value, { stream: true })
  // 解析并追加内容
}
```

**SSE 发送**：
```java
SseEmitter emitter = new SseEmitter(60000L)
contentStream.subscribe(
  content -> emitter.send(SseEmitter.event().data("data: " + json)),
  error -> emitter.completeWithError(error),
  () -> emitter.complete()
)
```

### 用户体验优化

1. **实时反馈**：内容逐字显示，体验更流畅
2. **代码美化**：代码块自动高亮，易于阅读
3. **格式丰富**：支持完整的 Markdown 语法
4. **自动滚动**：新内容自动滚动到可见区域
5. **错误处理**：网络错误或超时有明确提示

### 安装说明

**前端安装依赖**：
```bash
cd llm-frontend
npm install
```

新增的依赖会自动安装：
- marked：Markdown 解析
- highlight.js：代码高亮

### 变更原因

1. **Markdown 渲染**：AI 回复经常包含格式化内容（代码、列表、表格等），纯文本显示不够友好，需要渲染为富文本。

2. **流式输出**：传统的一次性返回需要等待完整响应，用户体验差。流式输出可以实时显示生成内容，类似 ChatGPT 的打字效果，大幅提升用户体验。

3. **代码高亮**：技术问答场景下经常涉及代码，语法高亮可以提高可读性。

---

---

---

## 2026-02-19 (更新7)

### 任务描述
修复流式/非流式接口设计问题，分离为两个独立的端点

### 问题描述

原来的设计将流式和非流式都放在同一个接口 `POST /api/v1/chat`，并通过 `stream` 参数控制。但是接口声明了 `produces = MediaType.TEXT_EVENT_STREAM_VALUE`，这会导致：

1. **Content-Type 固定**：响应头始终是 `text/event-stream`
2. **非流式响应异常**：非流式响应也被强制为 SSE 格式
3. **客户端解析困难**：前端需要根据参数判断如何解析响应

### 修复方案

**后端改造**：
1. **分离接口**：
   - `POST /api/v1/chat` - 非流式接口（`produces = MediaType.APPLICATION_JSON_VALUE`）
   - `POST /api/v1/chat/stream` - 流式接口（`produces = MediaType.TEXT_EVENT_STREAM_VALUE`）

2. **移除 stream 参数**：不再需要通过参数控制，由接口路径决定

3. **方法重命名**：
   - `chatStream()` 公开方法 → 调用 `streamChat()` 私有方法
   - 避免方法名冲突

**前端改造**：
1. **根据配置选择接口**：
   - 流式模式：调用 `/api/v1/chat/stream`
   - 非流式模式：调用 `/api/v1/chat`

2. **移除 stream 参数**：请求体中不再包含 `stream` 字段

### 修改文件

**后端**：
- `ChatController.java`：
  - 分离为两个 `@PostMapping` 方法
  - 不同的路径和 `produces` 类型
  - 私有方法 `streamChat()` 和 `chatNonStream()`

**前端**：
- `Chat.vue`：
  - `handleStreamSend()` 调用 `/chat/stream`
  - `handleNonStreamSend()` 调用 `/chat`
  - 移除请求体中的 `stream` 参数

### API 端点

**非流式对话**：
```
POST /api/v1/chat
Content-Type: application/json
Accept: application/json

Response: text/event-stream (SSE 格式，但一次性返回)
```

**流式对话**：
```
POST /api/v1/chat/stream
Content-Type: application/json
Accept: text/event-stream

Response: text/event-stream (SSE 格式，逐步返回)
```

### 设计优势

1. **职责清晰**：每个接口只做一件事
2. **类型正确**：Content-Type 与实际响应格式匹配
3. **易于扩展**：可以为流式和非流式添加不同的配置
4. **符合 RESTful**：使用路径区分不同的资源操作方式
5. **客户端简单**：根据 URL 就知道响应格式

### 变更原因

原设计违反了单一职责原则，一个接口试图处理两种完全不同的响应格式。通过分离接口，使得：
- 后端代码更清晰
- 前端调用更明确
- Content-Type 设置正确
- 符合 HTTP 协议规范

---

---

---

## 2026-02-19 (更新8)

### 任务描述
新增 Markdown 转图片功能模块，支持多种精美背景卡片模板

### 新增文件

**前端**：
- `MarkdownToImage.vue`：Markdown 转图片页面组件

### 修改文件

**前端**：
1. `router/index.js` - 添加 `/markdown-to-image` 路由
2. `App.vue` - 添加 Markdown 转图片菜单项，更新页面标题映射
3. `package.json` - 添加 html2canvas 依赖

### 核心功能

1. Markdown 编辑器：左侧实时编辑区，支持完整语法，代码高亮，一键加载示例
2. 实时预览：右侧实时渲染，使用 marked 解析，支持代码块、表格、引用等
3. 精美模板：渐变科技、优雅卡片、暗黑终端、清新自然、赛博朋克（5种）
4. 图片导出：使用 html2canvas，高清输出（scale: 2），一键下载 PNG

### 模板设计

- 渐变科技：紫色渐变背景，白色文字带阴影，半透明代码块
- 优雅卡片：浅灰渐变，标题带下划线，白色卡片式表格
- 暗黑终端：深色背景，绿色文字发光效果，等宽字体
- 清新自然：青绿渐变，多彩标题，半透明白色代码块
- 赛博朋克：霓虹渐变，青色标题带发光，扫描线效果，大写标题

### 技术实现
- Markdown 解析：marked 库，GFM 支持，代码高亮
- 代码高亮：highlight.js，Atom One Dark 主题
- 图片导出：html2canvas，2倍分辨率，透明背景
- 响应式布局：左右分栏，预览区可滚动，模板网格布局

### 变更原因

用户需要将 Markdown 文本转换为精美图片，用于社交媒体分享、文档截图等场景。通过提供多种精美模板和一键导出功能，提升内容创作效率和视觉效果。

---

---

---

## 2026-02-19 (更新9)

### 任务描述
为 Markdown 转图片功能添加复制到剪贴板能力，支持 PC 端和移动端

### 修改文件

**前端**：
- `MarkdownToImage.vue`：添加复制到剪贴板功能

### 核心功能

1. 使用 Clipboard API 直接复制图片，一键复制，无需下载再上传
2. 智能降级：优先 Clipboard API，不支持时自动降级为下载，权限被拒时提示并下载
3. 用户体验：复制和导出按钮分离，Loading 状态提示，成功/失败消息反馈

### 变更原因

用户反馈导出图片后还需要再次复制才能分享，操作繁琐。通过添加复制到剪贴板功能，实现一键复制，提升用户体验。

---

---

---

## 2026-02-19 (更新10)

### 任务描述
为整个应用添加移动端适配，包括导航栏和 Markdown 转图片模块

### 修改文件

**前端**：
1. `App.vue` - 添加移动端导航栏适配（折叠侧边栏、浮动菜单按钮、遮罩层）
2. `MarkdownToImage.vue` - 添加移动端布局适配（标签页切换、模板网格响应式、按钮图标化）

### 关键设计决策
- PC 端完整侧边栏（250px），移动端折叠（64px）只显示图标
- 移动端 Markdown 编辑/预览切换为标签页模式，避免左右滚动
- 768px 为 PC/移动端分界点

### 变更原因

用户需要在移动设备上使用应用，原有 PC 端布局在移动端显示不佳。

---

---

---

## 2026-02-19 (更新11)

### 任务描述
为所有页面模块统一添加移动端适配，使用 Element Plus 响应式栅格系统

### 修改文件

**前端**：`TemplateManagement.vue`、`ContentOptimization.vue`、`PromptTest.vue`、`ModelManagement.vue`、`PromptComparison.vue`

### 核心实现

统一使用 Element Plus 响应式栅格（`:xs="24" :sm="24" :md="12" :lg="12"`）和媒体查询统一移动端样式规范。

### 变更原因

通过统一使用 Element Plus 响应式栅格，避免自己实现响应式逻辑，降低维护成本，确保所有页面移动端体验一致。

---

---

---

## 2026-02-19 (更新12)

### 任务描述
修复所有表单在移动端显示不完整的问题，统一使用响应式表单布局

### 问题描述
所有页面表单使用固定 `label-width="120px"`，在移动端导致标签和内容挤在一起，用户无法正常使用。

### 修改文件

**前端**：`PromptTest.vue`、`ContentOptimization.vue`、`TemplateManagement.vue`、`ModelManagement.vue`、`PromptComparison.vue`、`Chat.vue`

### 核心解决方案

动态表单布局：`:label-width="isMobile ? '100%' : '120px'"` + `:label-position="isMobile ? 'top' : 'right'"`，移动端标签在上、内容在下，全宽显示。

### 变更原因

用户反馈移动端表单显示不完整，无法正常使用。通过 Element Plus 响应式表单属性实现移动端完美适配。

---

---

---

## 2026-02-18

### 任务描述
创建企业级 Spring AI + LangChain4j LLM 编排平台脚手架

### 创建的文件
1. **项目结构**
   - 根 pom.xml：多模块 Maven 项目配置
   - llm-domain：领域层模块
   - llm-application：应用层模块
   - llm-infrastructure：基础设施层模块
   - llm-api：API 层模块
   - llm-starter：启动模块

2. **领域层 (llm-domain)**
   - `LLMRequest.java`：LLM 请求模型
   - `LLMResponse.java`：LLM 响应模型
   - `Message.java`：对话消息模型
   - `TokenUsage.java`：Token 使用统计模型
   - `LLMProvider.java`：LLM 提供商接口
   - `ChainExecutor.java`：Chain 执行器接口
   - `ConversationMemoryRepository.java`：对话记忆仓储接口

3. **应用层 (llm-application)**
   - `LLMOrchestrationService.java`：LLM 编排服务，负责路由到合适的 Provider
   - `ConversationService.java`：对话管理服务
   - `ChatUseCase.java`：对话用例，编排完整对话流程

4. **基础设施层 (llm-infrastructure)**
   - `LLMConfiguration.java`：LLM 配置类，支持动态刷新
   - `OpenAIProvider.java`：OpenAI 提供商实现（使用 Spring AI）
   - `OllamaProvider.java`：Ollama 提供商实现（使用 LangChain4j）
   - `InMemoryConversationRepository.java`：内存对话仓储实现

5. **API 层 (llm-api)**
   - `ChatController.java`：对话 REST 控制器
   - `ChatRequest.java`：对话请求 DTO
   - `ChatResponse.java`：对话响应 DTO
   - `GlobalExceptionHandler.java`：全局异常处理器

6. **启动模块 (llm-starter)**
   - `LLMOrchestrationApplication.java`：Spring Boot 启动类
   - `application.yml`：应用配置文件

7. **文档**
   - `README.md`：项目说明文档
   - `.gitignore`：Git 忽略配置

### 核心设计决策

1. **架构选择**：采用 DDD-lite 分层架构
   - 领域层定义核心模型和接口
   - 应用层编排业务用例
   - 基础设施层实现外部集成
   - API 层处理协议转换
   - 依赖方向：api → application → domain ← infrastructure

2. **多 Provider 支持**
   - 使用策略模式，通过 `LLMProvider` 接口统一抽象
   - OpenAI 使用 Spring AI 集成
   - Ollama 使用 LangChain4j 集成
   - 支持根据模型名称自动路由到合适的 Provider

3. **配置管理**
   - 使用 `@RefreshScope` 支持动态刷新
   - 配置类和 Bean 都添加 `@RefreshScope` 确保刷新生效
   - 集中管理所有 LLM 相关配置

4. **依赖注入**
   - 全部使用构造函数注入，避免字段注入
   - 提高代码可测试性和可维护性

5. **对话管理**
   - 提供对话历史存储接口
   - 当前使用内存实现，生产环境可替换为 Redis
   - 支持获取最近 N 条消息用于上下文

### 变更原因
创建全新的企业级 LLM 编排平台脚手架，为后续 LLM 能力管理提供基础框架。采用清晰的分层架构和接口抽象，便于扩展和维护。

---

---

---

## 2026-02-18 (更新)

### 任务描述
添加内容优化功能模块，支持针对不同社交媒体平台（小红书、抖音、TikTok等）和不同风格进行文本内容优化

### 新增文件

1. **领域层扩展**
   - `PromptTemplate.java`：Prompt 模板接口
   - `ContentOptimizationRequest.java`：内容优化请求模型（包含平台、风格、内容类型枚举）
   - `ContentOptimizationResponse.java`：内容优化响应模型

2. **应用层**
   - `ContentOptimizationService.java`：内容优化服务，负责编排优化流程
   - `ContentOptimizationUseCase.java`：内容优化用例，包含请求校验和多版本生成

3. **基础设施层**
   - `ContentOptimizationPromptTemplate.java`：内容优化的 Prompt 模板实现

4. **API 层**
   - `ContentOptimizationController.java`：内容优化 REST 控制器
   - `ContentOptimizationRequestDTO.java`：请求 DTO
   - `ContentOptimizationResponseDTO.java`：响应 DTO

5. **文档**
   - `content-optimization-guide.md`：内容优化功能使用指南

### 核心设计决策

1. **平台适配**
   - 定义 5 个主流平台：小红书、抖音、TikTok、微博、微信公众号
   - 每个平台有独特的特点描述，用于指导 LLM 优化

2. **风格系统**
   - 支持 6 种内容风格：专业严谨、轻松随意、幽默风趣、情感共鸣、励志激励、潮流时尚
   - 每种风格有明确的描述，确保输出符合预期

3. **Prompt 工程**
   - 使用模板模式管理 Prompt
   - 结构化输出（JSON 格式），包含优化内容、标题建议、标签建议
   - 明确优化要求和输出格式

4. **多版本生成**
   - 支持一次生成 1-5 个不同版本
   - 便于用户对比选择最佳方案

5. **响应解析**
   - 使用 Jackson 解析 JSON 响应
   - 容错处理：解析失败时返回原始内容

### 变更原因
用户需要针对不同社交媒体平台优化文本内容，用于图文创作。通过 LLM 能力实现智能内容改写，提升内容质量和平台适配度。

---

---

---

## 2026-02-18 (更新2)

### 任务描述
将 Prompt 模板从代码硬编码迁移到 MySQL 数据库，支持动态管理和更新

### 新增文件

1. **领域层**
   - `PromptTemplateRepository.java`：Prompt 模板仓储接口

2. **基础设施层**
   - `PromptTemplateEntity.java`：Prompt 模板实体类
   - `PromptTemplateJpaRepository.java`：JPA Repository 接口
   - `PromptTemplateRepositoryImpl.java`：仓储实现，包含缓存
   - `DatabasePromptTemplate.java`：数据库模板实现
   - `CacheConfiguration.java`：Caffeine 缓存配置

3. **API 层**
   - `PromptTemplateController.java`：模板管理 REST 接口

4. **数据库脚本**
   - `init-database.sql`：数据库初始化脚本

5. **文档**
   - `prompt-template-management.md`：模板管理指南

### 修改文件

1. **llm-infrastructure/pom.xml**
   - 添加 MySQL 驱动依赖
   - 添加 Spring Data JPA 依赖
   - 添加 Caffeine 缓存依赖

2. **application.yml**
   - 添加数据源配置
   - 添加 JPA 配置

3. **ContentOptimizationService.java**
   - 从 PromptTemplateRepository 读取模板
   - 移除硬编码的模板依赖

### 核心设计决策

1. **数据库设计**
   - 使用 MySQL 存储模板
   - 支持版本管理和软删除
   - 记录创建人和更新人
   - 添加索引优化查询性能

2. **缓存策略**
   - 使用 Caffeine 本地缓存
   - 缓存时间：30 分钟
   - 最大容量：1000 个模板
   - 更新/删除时自动清空缓存

3. **Repository 模式**
   - 领域层定义接口
   - 基础设施层实现
   - 使用 JPA 简化数据访问

4. **模板管理 API**
   - 支持 CRUD 操作
   - 按名称或分类查询
   - RESTful 风格

5. **向后兼容**
   - 保留原有的 PromptTemplate 接口
   - 新增 DatabasePromptTemplate 实现
   - 平滑迁移，不影响现有功能

### 变更原因
Prompt 模板硬编码在代码中不够灵活，无法动态调整。迁移到数据库后，可以通过 API 或数据库直接管理模板，支持版本控制和团队协作，提升系统的可维护性和扩展性。

### 删除文件
- `ContentOptimizationPromptTemplate.java`：硬编码的模板类，已被数据库方案替代

---

---

---

## 2026-02-18 (更新3)

### 任务描述
新增 Prompt 测试功能，用于在前端页面验证不同模板和模型的输出效果

### 新增文件

1. **API 层**
   - `PromptTestController.java`：Prompt 测试 REST 控制器
   - `PromptTestRequestDTO.java`：测试请求 DTO
   - `PromptTestResponseDTO.java`：测试响应 DTO

2. **应用层**
   - `PromptTestUseCase.java`：Prompt 测试用例

3. **文档**
   - `prompt-test-guide.md`：Prompt 测试功能使用指南（含前端集成示例）

### 核心功能

1. **模板测试**
   - 选择数据库中的任意模板
   - 填写模板变量
   - 实时查看渲染后的 Prompt

2. **模型选择**
   - 支持 OpenAI 模型（GPT-4、GPT-3.5-turbo 等）
   - 支持 Ollama 本地模型（Llama、Mistral、Qwen 等）
   - 可自定义 temperature 和 maxTokens 参数

3. **结果展示**
   - 显示渲染后的完整 Prompt
   - 显示 LLM 输出结果
   - 显示 Token 使用情况
   - 显示执行耗时

4. **辅助功能**
   - 获取可用模型列表
   - 获取模板变量示例
   - 自动填充默认值

### API 端点

- `POST /api/v1/prompt-test` - 执行 Prompt 测试
- `GET /api/v1/prompt-test/models` - 获取可用模型列表
- `GET /api/v1/prompt-test/template-variables/{templateName}` - 获取模板变量示例

### 核心设计决策

1. **实时测试**
   - 直接调用 LLM API，返回真实结果
   - 显示完整的性能指标

2. **灵活配置**
   - 支持自定义所有 LLM 参数
   - 可选择不同的提供商和模型

3. **用户友好**
   - 提供模板变量示例
   - 自动填充默认值
   - 清晰的结果展示

4. **前端集成**
   - 提供 Vue 3 和 React 示例代码
   - RESTful API 设计，易于集成

### 使用场景

1. **模板调试**：开发新模板时快速验证效果
2. **模型对比**：对比不同模型的输出质量
3. **参数优化**：调整参数找到最佳配置
4. **变量测试**：测试不同变量值的影响

### 变更原因
用户需要在前端页面验证不同模板和模型的输出效果，方便调试和优化 Prompt。通过可视化的测试工具，提升开发效率和模板质量。

---

---

---

## 2026-02-18 (更新4)

### 任务描述
创建完整的前端工程，提供可视化界面调用后端 API

### 新增目录和文件

**前端项目结构** (`llm-frontend/`)：

1. **配置文件**
   - `package.json`：项目依赖和脚本配置
   - `vite.config.js`：Vite 构建配置和代理设置
   - `index.html`：HTML 入口文件
   - `.gitignore`：Git 忽略配置
   - `README.md`：前端项目说明文档

2. **源代码** (`src/`)
   - `main.js`：应用入口，配置 Vue、Router、Pinia、Element Plus
   - `App.vue`：根组件，包含侧边栏导航和布局

3. **路由** (`src/router/`)
   - `index.js`：路由配置，定义 4 个页面路由

4. **API 层** (`src/api/`)
   - `index.js`：封装所有后端 API 调用

5. **工具类** (`src/utils/`)
   - `request.js`：Axios 封装，包含请求/响应拦截器

6. **页面组件** (`src/views/`)
   - `PromptTest.vue`：Prompt 测试工具页面
   - `ContentOptimization.vue`：内容优化页面
   - `TemplateManagement.vue`：模板管理页面
   - `Chat.vue`：对话测试页面

### 技术栈

- **Vue 3**：使用 Composition API
- **Vite**：快速的开发构建工具
- **Element Plus**：Vue 3 UI 组件库
- **Vue Router**：路由管理
- **Pinia**：状态管理
- **Axios**：HTTP 客户端

### 核心功能

1. **Prompt 测试工具**
   - 选择模板和模型
   - 填写变量并实时渲染
   - 查看 LLM 输出和性能指标
   - 支持复制结果

2. **内容优化**
   - 输入原始内容
   - 选择平台、风格、类型
   - 生成 1-5 个优化版本
   - 显示标题、标签建议

3. **模板管理**
   - 列表展示所有模板
   - 创建、编辑、删除模板
   - 搜索和分类筛选

4. **对话测试**
   - 实时对话交互
   - 消息历史展示
   - 模型和提供商选择

### 核心设计决策

1. **现代化 UI**
   - 使用 Element Plus 组件库
   - 响应式布局
   - 深色侧边栏 + 浅色主内容区

2. **API 代理**
   - 开发环境通过 Vite 代理转发到后端
   - 避免跨域问题

3. **统一请求处理**
   - Axios 拦截器统一处理错误
   - 自动显示错误提示

4. **组件化设计**
   - 每个功能独立页面组件
   - 可复用的工具函数

5. **用户体验优化**
   - Loading 状态提示
   - 操作成功/失败反馈
   - 一键复制功能
   - 快捷键支持

### 启动方式

```bash
# 安装依赖
cd llm-frontend
npm install

# 启动开发服务器
npm run dev

# 访问 http://localhost:3000
```

### 变更原因
用户需要可视化界面来调用后端 API，方便测试和使用 LLM 编排平台的各项功能。通过完整的前端工程，提供友好的用户界面和良好的交互体验。

---

---

---

## 2026-02-18 (更新5)

### 任务描述
修复 llm-domain 模块中 Flux 类型无法解析的依赖问题

### 修改文件
- `llm-domain/pom.xml`：添加 reactor-core 依赖

### 问题描述
在 `LLMProvider.java` 接口中使用了 `Flux<String>` 类型作为流式调用的返回值，但 IDEA 提示无法解析 Flux 类型。

### 根本原因
`Flux` 类型来自 Project Reactor 库（`reactor-core`），但 llm-domain 模块的 pom.xml 中没有引入该依赖。虽然父 pom 中有 Spring Boot 依赖管理，但 domain 模块作为纯领域层，需要显式声明对 reactor-core 的依赖。

### 解决方案
在 `llm-domain/pom.xml` 的 dependencies 中添加：

```xml
<!-- Reactor Core (提供 Flux/Mono 响应式类型) -->
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-core</artifactId>
</dependency>
```

### 设计考虑
- reactor-core 版本由 Spring Boot Parent 管理，无需指定版本号
- 作为领域层接口定义的一部分，响应式类型依赖是合理的
- 不会引入过多传递依赖，保持领域层的轻量性

### 变更原因
修复编译错误，确保 LLMProvider 接口中的 Flux 类型能够正确解析，使项目可以正常编译和运行。

---

---

---

## 2026-02-18 (更新6)

### 任务描述
创建缺失的 PromptTestRequestDTO 类

### 新增文件
- `llm-api/src/main/java/com/enterprise/llm/api/dto/PromptTestRequestDTO.java`

### 问题描述
PromptTestController 中引用了 PromptTestRequestDTO，但该类尚未创建，导致编译错误。

### 解决方案
创建 PromptTestRequestDTO 类，包含以下字段：
- templateName：模板名称（必填）
- variables：模板变量 Map（必填）
- model：模型名称（必填）
- provider：提供商（可选）
- temperature：温度参数（可选）
- maxTokens：最大 token 数（可选）

### 设计决策
- 使用 Jakarta Validation 注解进行参数校验
- 使用 Lombok 简化代码
- 遵循项目中其他 DTO 的命名和结构规范

### 变更原因
补充缺失的 DTO 类，解决 PromptTestController 中的编译错误，使 Prompt 测试功能完整可用。

---

---

---

## 2026-02-18 (更新7)

### 任务描述
修复 DDD-lite 架构中的依赖问题，避免 API 层直接依赖 Infrastructure 层

### 问题描述
PromptTemplateController 中直接引用了 DatabasePromptTemplate（来自 infrastructure 层），违反了 DDD-lite 架构的依赖原则。API 层不应该直接依赖 Infrastructure 层的具体实现。

### 解决方案

**架构原则：**
- API 层 → Application 层 → Domain 层
- Infrastructure 层 → Domain 层
- API 层不应该直接依赖 Infrastructure 层

**具体修改：**

1. **扩展 Domain 接口**
   - 在 `PromptTemplate` 接口中添加 `getTemplateContent()`、`getCategory()`、`getDescription()` 方法
   - 使接口包含完整的模板信息

2. **引入工厂模式**
   - 创建 `PromptTemplateFactory` 接口（Domain 层）
   - 创建 `DatabasePromptTemplateFactory` 实现（Infrastructure 层）
   - 通过工厂创建模板实例，避免直接依赖具体类

3. **创建 Application 层 UseCase**
   - 创建 `PromptTemplateManagementUseCase`
   - 封装模板管理的业务逻辑
   - 提供 `TemplateInfo` DTO 和 `TemplateCreateRequest` DTO

4. **简化 Controller**
   - Controller 只依赖 UseCase
   - 不再直接使用 Repository 和具体实现类
   - 保持 API 层的简洁性

### 新增文件
- `llm-domain/src/main/java/com/enterprise/llm/domain/service/PromptTemplateFactory.java`
- `llm-infrastructure/src/main/java/com/enterprise/llm/infrastructure/prompt/DatabasePromptTemplateFactory.java`
- `llm-application/src/main/java/com/enterprise/llm/application/usecase/PromptTemplateManagementUseCase.java`

### 修改文件
- `llm-domain/src/main/java/com/enterprise/llm/domain/service/PromptTemplate.java`：扩展接口方法
- `llm-api/src/main/java/com/enterprise/llm/api/controller/PromptTemplateController.java`：使用 UseCase，移除对 Infrastructure 的依赖

### 核心设计决策

1. **依赖方向正确性**
   - 严格遵循 DDD-lite 分层架构
   - 通过接口和工厂模式解耦层级依赖

2. **职责分离**
   - Domain 层：定义接口和契约
   - Application 层：编排业务逻辑
   - Infrastructure 层：提供具体实现
   - API 层：处理协议转换

3. **可测试性**
   - UseCase 可以独立测试
   - Controller 只需 mock UseCase
   - 各层职责清晰，易于维护

### 变更原因
修复架构违规问题，确保各层依赖方向正确。在 DDD-lite 架构中，子模块之间的依赖需要在各自的 pom.xml 中明确配置，但更重要的是要遵循正确的依赖方向，避免跨层直接依赖具体实现。

---

---
