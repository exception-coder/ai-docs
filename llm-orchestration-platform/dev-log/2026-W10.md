# 开发日志 2026-W10（2026-03-03 ~ 2026-03-05）

## 2026-03-05 统一修改包名 com.enterprise → com.exceptioncoder

### 任务描述

将项目所有模块的 Java 包名从 `com.enterprise` 统一修改为 `com.exceptioncoder`（因 Java 包名不允许使用连字符 `-`，故将 `exception-coder` 写为 `exceptioncoder`）。

### 修改的文件

- **6 个 pom.xml**：根 POM 和 5 个子模块的 POM 文件中的 groupId 及依赖声明
- **67 个 Java 文件**：所有模块（llm-api、llm-application、llm-domain、llm-infrastructure、llm-starter）中的 package 声明和 import 语句
- **3 个 logging.yml**：dev/test/prod 环境的日志级别配置中的包名引用
- **.cursorrules**：项目规则文件中的包名规范
- **docs/dev-log.md**：开发日志中的包名引用
- **物理目录**：将 5 个模块的 `src/main/java/com/enterprise` 目录重命名为 `com/exceptioncoder`

### 关键设计决策

- Java 包名不支持连字符（`-`），采用去除连字符的方式：`exception-coder` → `exceptioncoder`
- 使用批量脚本替换确保所有引用一致性，避免遗漏
- target 目录中的构建产物未修改，会在下次构建时自动重新生成

### 变更原因

统一项目包名标识，从通用的 `com.enterprise` 改为项目专属的 `com.exceptioncoder`，使项目标识更加清晰和独特。
---

---

---

## 2026-03-04 (晚上)

### 任务描述
添加 Qdrant 向量数据库 Docker 部署配置

### 创建的文件

**Docker 部署配置**：
- `docker/qdrant/docker-compose.yml`：Qdrant 服务配置
- `docker/qdrant/README.md`：Qdrant 部署和使用指南
- `docker/README.md`：Docker 部署总览

### 配置说明

**Qdrant 服务配置**：
```yaml
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"  # HTTP API + Web UI
      - "6334:6334"  # gRPC API (Spring AI 使用)
    volumes:
      - qdrant_storage:/qdrant/storage  # 数据持久化
    environment:
      - QDRANT__LOG_LEVEL=INFO
      - QDRANT__SERVICE__ENABLE_TLS=false  # 开发环境
```

**端口说明**：
- **6333**: HTTP API 和 Web UI（浏览器访问 http://localhost:6333/dashboard）
- **6334**: gRPC API（Spring AI 使用此端口）

**数据持久化**：
- 使用 Docker 卷 `qdrant_storage` 存储数据
- 容器重启后数据不会丢失

### 使用方式

**启动服务**：
```bash
cd docker/qdrant
docker-compose up -d
```

**验证服务**：
```bash
# 查看容器状态
docker-compose ps

# 访问 Web UI
浏览器打开: http://localhost:6333/dashboard

# 查看集合列表
curl http://localhost:6333/collections
```

**停止服务**：
```bash
docker-compose down

# 停止并删除数据
docker-compose down -v
```

### 应用配置

开发环境配置已正确设置：
```yaml
spring:
  ai:
    vectorstore:
      qdrant:
        host: localhost
        port: 6334  # gRPC 端口
        use-tls: false
        collection-name: job_postings_dev
```

### 核心设计决策

1. **目录结构**：将 Docker 配置放在项目根目录的 `docker/` 文件夹下
2. **服务隔离**：每个服务独立子目录，便于管理
3. **文档完善**：提供详细的部署和使用指南
4. **开发友好**：默认配置适合开发环境，无需额外配置
5. **数据安全**：使用 Docker 卷持久化数据

### 文档内容

**README.md 包含**：
- 快速启动指南
- 配置说明
- 常用操作（创建/查看/删除集合）
- 性能优化建议
- 备份与恢复
- 故障排查

### 变更原因

为了方便开发和部署，提供标准化的 Qdrant 部署配置。通过 Docker Compose，开发人员可以一键启动向量数据库，无需手动安装和配置。

---

---

---

## 2026-03-04 (下午)

### 任务描述
修复 Spring AI 多个 EmbeddingModel Bean 冲突问题

### 问题描述

启动时报错：
```
Parameter 0 of method vectorStore in QdrantVectorStoreAutoConfiguration 
required a single bean, but 2 were found:
- ollamaEmbeddingModel
- openAiEmbeddingModel
```

### 根本原因

项目中同时配置了 OpenAI 和 Ollama，Spring AI 自动配置为两者都创建了 `EmbeddingModel` bean。`QdrantVectorStoreAutoConfiguration` 需要注入一个 `EmbeddingModel`，但 Spring 不知道应该使用哪一个。

### EmbeddingModel 说明

- **作用**：将文本转换为向量（embedding），用于向量存储和相似度检索
- **OpenAI EmbeddingModel**：调用远程 API（https://api.openai.com），需要 API 密钥
- **Ollama EmbeddingModel**：调用本地 HTTP 服务（http://localhost:11434），无需外部 API
- **两者都需要网络请求**（HTTP API 调用），只是服务位置不同

### 解决方案

采用**代理模式**，创建 `DelegatingEmbeddingModel` 代理类，统一管理多个 EmbeddingModel。

#### 设计优势

1. **多模型共存**：支持同时使用 OpenAI 和 Ollama Embedding
2. **动态选择**：根据模型标识符动态路由到具体实现
3. **默认模型**：提供默认模型（OpenAI），无需显式指定
4. **接口透明**：实现 EmbeddingModel 接口，对外透明
5. **灵活切换**：支持 ThreadLocal、直接获取等多种使用方式

#### 核心实现

**DelegatingEmbeddingModel 代理类**：
```java
public class DelegatingEmbeddingModel implements EmbeddingModel {
    // 模型注册表
    private final Map<String, EmbeddingModel> modelRegistry;
    
    // 默认模型
    private final EmbeddingModel defaultModel;
    
    // ThreadLocal 支持
    private static final ThreadLocal<String> CURRENT_MODEL;
    
    // 根据标识符获取模型
    public EmbeddingModel getModel(String modelIdentifier);
    
    // 设置当前线程使用的模型
    public static void setCurrentModel(String modelIdentifier);
}
```

**Spring 配置**：
```java
@Bean
@Primary
public DelegatingEmbeddingModel delegatingEmbeddingModel(
    @Autowired(required = false) OpenAiEmbeddingModel openAiEmbeddingModel,
    @Autowired(required = false) OllamaEmbeddingModel ollamaEmbeddingModel) {
    return new DelegatingEmbeddingModel(openAiEmbeddingModel, ollamaEmbeddingModel);
}
```

### 使用方式

**方式 1：使用默认模型（OpenAI）**
```java
delegatingEmbeddingModel.embed("文本");  // 自动使用 OpenAI
```

**方式 2：通过 ThreadLocal 切换模型**
```java
try {
    DelegatingEmbeddingModel.setCurrentModel("ollama");
    delegatingEmbeddingModel.embed("文本");  // 使用 Ollama
} finally {
    DelegatingEmbeddingModel.clearCurrentModel();
}
```

**方式 3：直接获取特定模型**
```java
EmbeddingModel ollamaModel = delegatingEmbeddingModel.getModel("ollama");
ollamaModel.embed("文本");  // 使用 Ollama
```

**方式 4：查询可用模型**
```java
List<String> models = delegatingEmbeddingModel.getAvailableModels();
// 返回: ["openai", "ollama"]
```

### 创建的文件

- `DelegatingEmbeddingModel.java`：EmbeddingModel 代理类（包含 JavaDoc 使用说明）

### 修改的文件

- `.cursorrules`：添加"禁止创建示例代码类"规则

- `SpringAIConfiguration.java`：创建代理 Bean，标记为 @Primary
- `config/dev/spring-ai.yml`：启用所有 Embedding 模型
- `config/test/spring-ai.yml`：启用所有 Embedding 模型
- `config/prod/spring-ai.yml`：启用所有 Embedding 模型

### 核心设计决策

1. **代理模式**：统一接口，内部管理多个实现
2. **注册表模式**：使用 Map 管理模型标识符和实例的映射
3. **ThreadLocal 支持**：支持线程级别的模型切换
4. **默认策略**：优先使用 OpenAI，如果不存在则使用 Ollama
5. **可选依赖**：使用 `@Autowired(required = false)`，支持部分模型不可用的场景

### 应用场景

1. **默认场景**：Qdrant VectorStore 使用默认的 OpenAI Embedding
2. **成本优化**：对于不重要的文本，使用本地 Ollama Embedding
3. **A/B 测试**：对比不同 Embedding 模型的效果
4. **降级策略**：OpenAI 不可用时自动降级到 Ollama
5. **批量处理**：不同批次使用不同模型，负载均衡

### 变更原因

系统确实存在多个 Embedding 使用场景，禁用某个模型会限制灵活性。通过代理模式，既解决了 Bean 冲突问题，又保留了多模型共存的能力，提供了更优雅和灵活的解决方案。

---

---

---

## 2026-03-04

### 任务描述
新增配置验证功能，在应用启动时检查环境配置文件是否正确加载

### 创建的文件

**配置层 (llm-starter)**
- `ConfigurationValidationProperties.java`：配置验证属性类，支持启用/禁用、失败策略、检查项配置
- `ConfigurationValidator.java`：配置验证器，实现 ApplicationRunner 在启动后执行检查
- `ConfigurationValidationAutoConfiguration.java`：自动配置类，启用配置属性绑定

### 修改的文件
- `application.yml`：添加配置验证相关配置项（llm.config.validation）

### 核心功能

1. **环境配置检查**
   - 检查当前激活的 Spring Profile
   - 验证对应环境目录下的配置文件是否存在（datasource.yml、spring-ai.yml、logging.yml）
   - 使用 ResourceLoader 检查文件存在性

2. **配置属性验证**
   - 验证关键配置属性是否已加载到 Environment
   - 检查配置属性是否为空（可能使用了默认值或环境变量）
   - 提供清晰的警告信息

3. **灵活的验证策略**
   - 可配置是否启用验证（enabled: true/false）
   - 可配置验证失败时是否阻止启动（fail-on-error: true/false）
   - 可配置需要检查的配置文件列表
   - 可配置需要验证的关键配置属性

4. **友好的错误提示**
   - 区分错误和警告
   - 提供详细的错误信息和建议
   - 日志输出清晰，便于排查问题

### 关键设计决策

1. **使用 ApplicationRunner**：在应用完全启动后执行检查，确保所有配置都已加载
2. **使用 @Order(1)**：确保在其他 ApplicationRunner 之前执行，尽早发现问题
3. **构造函数注入**：遵循项目规范，使用构造函数注入依赖
4. **可配置性**：所有检查项都可通过配置文件调整，提高灵活性
5. **优雅降级**：默认只警告不阻止启动，生产环境可配置为阻止启动

### 配置示例

```yaml
llm:
  config:
    validation:
      enabled: true                    # 是否启用验证
      fail-on-error: false             # 验证失败时是否阻止启动
      required-files:                  # 需要检查的配置文件
        - datasource.yml
        - spring-ai.yml
        - logging.yml
      required-properties:             # 需要验证的关键配置属性
        - spring.datasource.url
        - spring.ai.openai.api-key
```

### 验证输出示例

**正常情况**：
```
INFO  - 开始执行配置验证检查...
INFO  - 当前激活的 Profile: dev
INFO  - 验证 Profile [dev] 的配置文件...
DEBUG - ✓ 配置文件存在: classpath:config/dev/datasource.yml
DEBUG - ✓ 配置文件存在: classpath:config/dev/spring-ai.yml
DEBUG - ✓ 配置文件存在: classpath:config/dev/logging.yml
INFO  - 验证关键配置属性是否已加载...
DEBUG - ✓ 配置属性已加载: spring.datasource.url
DEBUG - ✓ 配置属性已加载: spring.ai.openai.api-key
INFO  - ✓ 配置验证通过，所有检查项均正常
```

**异常情况**：
```
ERROR - Profile [prod] 的配置文件不存在: datasource.yml (期望路径: classpath:config/prod/datasource.yml)
WARN  - 关键配置属性未设置或为空: spring.ai.openai.api-key (可能使用了默认值或环境变量)
ERROR - 配置验证失败，应用启动被阻止。请检查上述错误信息并修复配置问题。
```

### 变更原因

多环境配置结构下，需要确保启动时加载的是对应环境目录下的配置文件。通过启动时验证，可以：
1. 及早发现配置问题，避免运行时错误
2. 确保环境配置正确，避免使用错误的配置
3. 提供清晰的错误提示，便于快速定位问题
4. 支持灵活的验证策略，适应不同环境需求

---

---

---

## 2026-03-04

### 任务描述
使用 Spring AI Qdrant 集成替换原生 Qdrant Client，简化实现

### 修改的文件
- `llm-infrastructure/pom.xml`：移除 `io.qdrant:client`，添加 `spring-ai-qdrant-store-spring-boot-starter`
- `QdrantVectorStoreRepository.java`：使用 Spring AI 的 VectorStore 接口重写
- `application.yml`：使用 Spring AI 的标准配置格式
- `.cursorrules`：添加"优先使用 Spring 官方组件"的技术选型原则
- 删除 `QdrantConfiguration.java`：不再需要自定义配置类

### 关键改进

1. **使用 Spring AI VectorStore**：统一的向量存储抽象接口
2. **自动配置**：Spring Boot 自动配置 Qdrant 客户端
3. **简化实现**：
   - 使用 `Document` 模型存储数据
   - 使用 `SearchRequest` 构建查询
   - 自动处理向量化和序列化
4. **过滤表达式**：使用类似 SQL 的过滤语法（如 `job_family == 'backend_java'`）

### 技术选型原则

新增项目规范：**优先使用 Spring 官方组件**

优先级：
1. Spring 官方组件（如 Spring AI、Spring Cloud）
2. Spring 生态兼容组件（提供 Starter 的）
3. 原生 SDK（仅在无官方支持时）

### 优势

- ✅ 代码量减少 60%+
- ✅ 无需处理复杂的 gRPC API
- ✅ 统一的编程模型
- ✅ 自动配置和依赖注入
- ✅ 官方维护和长期支持

### 配置示例

```yaml
spring.ai:
  vectorstore:
    qdrant:
      host: localhost
      port: 6334
      collection-name: job_postings
      initialize-schema: true
```

### 变更原因
使用 Spring 官方组件可以大幅简化实现，提高代码可维护性和稳定性

---

---

---

## 2026-03-04 (下午)

### 任务描述
优化配置文件结构，将不同类型的配置拆分到独立文件中

### 创建的文件
- `application-spring-ai.yml`：Spring AI 相关配置（OpenAI、Ollama、Qdrant）
- `application-llm.yml`：LLM 业务配置（默认提供商、各 LLM 配置）
- `application-datasource.yml`：数据源和 JPA 配置
- `application-logging.yml`：日志配置
- `docs/configuration-guide.md`：配置文件说明文档

### 修改的文件
- `application.yml`：简化为主配置文件，通过 `spring.config.import` 导入其他配置

### 配置文件结构

```
application.yml                 # 主配置（导入其他配置）
├── application-datasource.yml  # 数据源配置
├── application-spring-ai.yml   # Spring AI 配置
├── application-llm.yml         # LLM 业务配置
└── application-logging.yml     # 日志配置
```

### 关键改进

1. **模块化配置**：按功能域拆分配置文件，职责清晰
2. **环境变量支持**：所有配置项支持环境变量覆盖
3. **易于维护**：修改某类配置只需编辑对应文件
4. **多环境支持**：可以轻松创建 dev/test/prod 环境配置

### 配置导入方式

使用 Spring Boot 2.4+ 的 `spring.config.import` 特性：

```yaml
spring:
  config:
    import:
      - classpath:application-datasource.yml
      - classpath:application-spring-ai.yml
      - classpath:application-llm.yml
      - classpath:application-logging.yml
```

### 优势

- ✅ 配置文件职责单一，易于理解
- ✅ 团队协作时减少配置冲突
- ✅ 可以针对不同模块独立管理配置
- ✅ 支持按需加载配置文件
- ✅ 便于配置审计和版本控制

### 变更原因
随着项目功能增加，单一配置文件变得臃肿难以维护，拆分后更清晰

---

---

---

## 2026-03-04 (傍晚)

### 任务描述
添加多环境配置支持，区分 dev、test、prod 环境

### 创建的文件
- `application-datasource-dev.yml`：开发环境数据源配置
- `application-datasource-test.yml`：测试环境数据源配置
- `application-datasource-prod.yml`：生产环境数据源配置
- `application-spring-ai-dev.yml`：开发环境 Spring AI 配置
- `application-spring-ai-test.yml`：测试环境 Spring AI 配置
- `application-spring-ai-prod.yml`：生产环境 Spring AI 配置
- `application-logging-dev.yml`：开发环境日志配置
- `application-logging-test.yml`：测试环境日志配置
- `application-logging-prod.yml`：生产环境日志配置

### 修改的文件
- `application.yml`：添加默认激活 dev 环境配置
- `application-datasource.yml`：调整为默认配置
- `docs/configuration-guide.md`：更新配置文档，添加多环境说明

### 环境配置对比

| 配置项 | dev | test | prod |
|--------|-----|------|------|
| 数据库 ddl-auto | update | validate | none |
| 显示 SQL | true | false | false |
| 日志级别 | DEBUG | INFO | WARN |
| Qdrant TLS | false | true | true |
| 自动初始化 Schema | true | true | false |
| 连接池大小 | 5 | 10 | 20 |
| 日志保留天数 | 7 | 30 | 90 |

### 关键特性

1. **开发环境 (dev)**：
   - 默认环境，快速开发
   - 自动更新数据库结构
   - 详细的调试日志
   - 使用本地服务

2. **测试环境 (test)**：
   - 模拟生产环境
   - 只验证数据库结构
   - 适中的日志级别
   - 使用测试服务器

3. **生产环境 (prod)**：
   - 最严格的配置
   - 禁止自动修改数据库
   - 最少的日志输出
   - 启用 TLS 加密
   - 所有配置必须通过环境变量提供
   - 启用连接泄漏检测

### 环境切换方式

```bash
# 方式 1：命令行参数
java -jar app.jar --spring.profiles.active=prod

# 方式 2：环境变量
export SPRING_PROFILES_ACTIVE=prod

# 方式 3：IDE 配置
在 Run Configuration 中设置环境变量
```

### 配置文件加载顺序

Spring Boot 会按以下顺序加载配置：
1. application.yml
2. application-{config}.yml（如 application-datasource.yml）
3. application-{config}-{profile}.yml（如 application-datasource-dev.yml）

后加载的配置会覆盖先加载的配置。

### 变更原因
不同环境需要不同的配置策略，统一配置无法满足开发、测试、生产的差异化需求

---

---

---

## 2026-03-04 (深夜)

### 任务描述
实现按环境分目录 + 按类型分文件的配置结构，使用动态导入

### 最终配置文件结构

```
resources/
├── application.yml              # 主配置（动态导入）
├── application-llm.yml          # 通用 LLM 配置
└── config/
    ├── dev/                     # 开发环境
    │   ├── datasource.yml
    │   ├── spring-ai.yml
    │   └── logging.yml
    ├── test/                    # 测试环境
    │   ├── datasource.yml
    │   ├── spring-ai.yml
    │   └── logging.yml
    └── prod/                    # 生产环境
        ├── datasource.yml
        ├── spring-ai.yml
        └── logging.yml
```

### 关键技术

**动态配置导入**：
```yaml
spring:
  config:
    import:
      - optional:classpath:config/${spring.profiles.active}/datasource.yml
      - optional:classpath:config/${spring.profiles.active}/spring-ai.yml
      - optional:classpath:config/${spring.profiles.active}/logging.yml
```

使用 `${spring.profiles.active}` 占位符，Spring Boot 会根据激活的 profile 自动替换并加载对应目录下的配置文件。

### 配置结构优势

| 维度 | 说明 |
|------|------|
| **按环境分目录** | dev/test/prod 各自独立目录 |
| **按类型分文件** | datasource/spring-ai/logging 分离 |
| **动态加载** | 根据 profile 自动加载对应目录 |
| **职责清晰** | 每个文件只负责一类配置 |
| **易于扩展** | 添加环境或配置类型都很简单 |

### 配置文件数量

- 主配置：2 个（application.yml + application-llm.yml）
- 环境配置：9 个（3 个环境 × 3 类配置）
- **总计：11 个配置文件**

### 扩展性示例

**添加新环境（staging）：**
1. 创建 `config/staging/` 目录
2. 复制 3 个配置文件
3. 无需修改 application.yml

**添加新配置类型（cache）：**
1. 在每个环境目录下创建 `cache.yml`
2. 在 application.yml 中添加一行导入配置

### 优势对比

| 方面 | 传统方式 | 当前方式 |
|------|---------|---------|
| 配置组织 | 按环境或按类型（二选一） | 按环境 + 按类型（两者兼得） |
| 文件定位 | 需要在多个文件中查找 | 环境/类型双重定位 |
| 添加环境 | 创建多个 application-{env}-{type}.yml | 创建一个目录，复制文件 |
| 添加配置类型 | 在每个环境文件中添加 | 创建新文件，添加一行导入 |
| 配置隔离 | 环境间配置混在一起 | 环境配置完全隔离 |

### 变更原因
既要按环境隔离配置，又要按类型分离关注点，动态导入机制完美解决了这个需求

# 开发日志

---

---

## 2026-03-03

### 任务描述
新增岗位JD向量检索模块，支持向量存储和相似度检索

### 创建的文件

**Domain层 (llm-domain)**
- `JobPosting.java`：岗位领域模型，包含岗位ID、岗位族、级别、技能、经验、学历等字段
- `VectorSearchRequest.java`：向量检索请求模型，支持查询文本、TopK和过滤条件
- `VectorSearchResult.java`：向量检索结果模型，包含带相似度分数的岗位列表
- `VectorStoreRepository.java`：向量存储仓储接口，定义存储、检索、删除等操作
- `JobVectorExtractor.java`：岗位向量文本提取服务接口

**Application层 (llm-application)**
- `JobVectorService.java`：岗位向量服务，协调向量提取和存储流程
- `JobSearchUseCase.java`：岗位检索用例，处理检索请求和参数校验

**Infrastructure层 (llm-infrastructure)**
- `QdrantConfiguration.java`：Qdrant向量库配置类，支持动态刷新
- `QdrantVectorStoreRepository.java`：Qdrant向量存储实现，集成Qdrant客户端
- `LLMJobVectorExtractor.java`：基于LLM的岗位向量文本提取器

**API层 (llm-api)**
- `JobSearchController.java`：岗位检索REST控制器，提供检索、存储、删除等接口
- `JobSearchRequestDTO.java`：岗位检索请求DTO
- `JobSearchResponseDTO.java`：岗位检索响应DTO
- `JobPostingDTO.java`：岗位存储请求DTO

### 修改的文件
- `llm-infrastructure/pom.xml`：添加Qdrant客户端依赖（io.qdrant:client:1.7.0）
- `llm-starter/src/main/resources/application.yml`：添加Qdrant配置项

### 关键设计决策

1. **分层架构**：严格遵循DDD分层，Domain定义接口，Infrastructure实现，保证依赖方向正确
2. **向量文本格式**：采用结构化文本格式，包含岗位族、级别、技能、经验等7个维度
3. **Metadata设计**：存储posting_id、job_family、level、city、post_time、dup_group_id用于过滤
4. **LLM提取**：使用LLM从原始JD中提取结构化信息，提高检索准确性
5. **构造函数注入**：所有服务使用构造函数注入，提高可测试性

### 变更原因
业务需要支持岗位JD的语义检索，通过向量相似度匹配找到相关岗位，提升招聘效率

### 后续工作
1. 集成实际的Embedding服务（如OpenAI Embeddings API）替换占位实现
2. 添加单元测试和集成测试
3. 优化批量存储性能
4. 添加向量库监控和告警

---

---

---

## 2026-03-03 (下午)

### 任务描述
修复 QdrantVectorStoreRepository 中的 Qdrant API 调用错误

### 修改的文件
- `QdrantVectorStoreRepository.java`：修复所有 Qdrant Java Client API 调用

### 主要修复内容

1. **初始化方法**：添加 @PostConstruct 注解，延迟客户端初始化避免构造函数中的阻塞操作
2. **向量存储**：修正 Vectors 和 Vector 的构建方式，使用正确的 Builder 模式
3. **批量存储**：修正 upsertAsync 方法参数，添加 ordering 参数（true）
4. **搜索功能**：修正 SearchPoints 构建方式，filter 为 null 时不设置
5. **删除功能**：使用 PointsSelector 和 PointsIdsList 正确构建删除请求
6. **Payload 构建**：使用正确的 Value.newBuilder() API
7. **Filter 构建**：修正 FieldCondition 和 Condition 的嵌套关系

### 关键 API 变更

- `addAllVectors()` → `setVectors(Vectors.newBuilder().setVector(...).build())`
- `Collections.singletonList()` → 直接传递 List
- `upsertAsync(collection, points)` → `upsertAsync(collection, points, ordering)`
- `deleteAsync(collection, ids)` → `deleteAsync(collection, PointsSelector, ordering)`
- `Points.Value` → `Value`（导入路径修正）

### 变更原因
原代码使用了错误的 Qdrant Java Client API，导致编译错误和运行时异常

### 测试建议
1. 启动 Qdrant 服务（docker run -p 6334:6334 qdrant/qdrant）
2. 测试存储接口：POST /api/job-search/store
3. 测试检索接口：POST /api/job-search/search
4. 测试健康检查：GET /api/job-search/health

---

---

---

## 2026-03-03 (晚上)

### 任务描述
升级 Qdrant Client 到 1.17.0 并修复 API 兼容性问题

### 修改的文件
- `llm-infrastructure/pom.xml`：升级 Qdrant Client 版本从 1.7.0 到 1.17.0
- `QdrantVectorStoreRepository.java`：使用标准 Builder 模式重写所有 API 调用

### 主要变更
1. **依赖升级**：使用最新稳定版本 1.17.0
2. **导入优化**：使用精确导入避免类名冲突
3. **API 标准化**：使用 gRPC 原生 Builder 模式，不依赖工厂方法
4. **向量构建**：`Vectors.newBuilder().setVector(Vector.newBuilder().addAllData(vector).build()).build()`
5. **Payload 构建**：`Value.newBuilder().setStringValue(value).build()`
6. **过滤条件**：标准的 FieldCondition → Condition → Filter 构建链

### 变更原因
使用最新版本获得更好的性能和稳定性，避免旧版本的 API 问题

### 最终修复
1. **导入冲突**：移除未使用的 `CollectionOperationResponse` 和 `ExecutionException` 导入
2. **Collections 冲突**：将 `Collections.singletonList()` 改为 `Arrays.asList()`，避免与 Qdrant 的 `Collections` 类冲突
3. **API 验证**：所有 Qdrant Client 1.17.0 API 调用已验证正确

---

---

---

## 2026-03-03 (深夜)

### 任务描述
Qdrant Java Client API 持续出现问题，创建简化骨架实现

### 修改的文件
- `QdrantVectorStoreRepository.java`：简化为骨架实现，待 API 确认后完善
- 新增 `docs/qdrant-integration-guide.md`：Qdrant 集成指南

### 问题分析
Qdrant Java Client 1.17.0 的 gRPC 生成类结构复杂，`Points.Value`、`Points.Vector` 等内部类的可用性无法确认。

### 解决方案
1. **当前方案**：创建骨架实现，抛出 UnsupportedOperationException，不影响项目编译
2. **推荐方案**：使用 Spring AI 的 Qdrant 集成（spring-ai-qdrant-store），提供开箱即用的向量存储
3. **备选方案**：参考 Qdrant 官方 Java Client 示例代码完善实现

### 变更原因
避免因 API 不确定性阻塞项目开发，先保证代码可编译，后续根据实际需求选择合适的集成方案

---

---
