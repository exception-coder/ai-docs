# Spring AI 版本踩坑日记

> 记录时间：2026-04-08
> 涉及项目：llm-orchestration-platform
> 作者：张凯

---

## 背景

项目最初使用 `spring-ai 1.0.0-M4`（里程碑版）+ `spring-ai-alibaba 1.0.0.4` + `spring-boot 3.2.2`。随着功能迭代（新增画像向量化模块、Qdrant 多 collection 支持），启动时开始频繁报错。本文记录排查过程和最终解决方案。

---

## 坑 1：chatClientBuilderConfigurer Bean 重复注册

### 现象

```
The bean 'chatClientBuilderConfigurer', defined in class path resource
[org/springframework/ai/model/chat/client/autoconfigure/ChatClientAutoConfiguration.class],
could not be registered. A bean with that name has already been defined in class path resource
[org/springframework/ai/autoconfigure/chat/client/ChatClientAutoConfiguration.class]
and overriding is disabled.
```

### 原因

`spring-ai 1.0.0-M4` 的 auto-configuration 在 `org.springframework.ai.autoconfigure.chat.client` 包下，而 `spring-ai-alibaba 1.0.0.4` 编译依赖的是 `spring-ai 1.0.1`（GA），其 auto-configuration 重构到了 `org.springframework.ai.model.chat.client.autoconfigure` 包下。

两个版本的类同时存在于 classpath，注册了同名 Bean `chatClientBuilderConfigurer`，Spring Boot 默认禁止覆盖，直接启动失败。

### 临时方案（不推荐）

```yaml
spring:
  main:
    allow-bean-definition-overriding: true
```

能跑起来，但掩盖了版本不一致的根因，后续可能出现更隐蔽的运行时问题。

### 根治方案

将 `spring-ai` 升级到 `1.0.1`（GA），与 `spring-ai-alibaba 1.0.0.4` 对齐。同时升级 `spring-boot` 到 `3.4.5`（spring-ai 1.0.1 依赖 Spring Framework 6.2.x）。

---

## 坑 2：spring-ai artifact 重命名（M4 → 1.0.1）

### 现象

升级 `spring-ai` 到 `1.0.1` 后，Maven 报依赖找不到：

```
Could not find artifact org.springframework.ai:spring-ai-qdrant-store-spring-boot-starter
Could not find artifact org.springframework.ai:spring-ai-ollama
```

### 原因

spring-ai 从 `1.0.0-M7` 开始统一重命名了 artifact，遵循新命名规范：

| 旧名称（M4） | 新名称（1.0.1 GA） |
|--------------|-------------------|
| `spring-ai-qdrant-store-spring-boot-starter` | `spring-ai-starter-vector-store-qdrant` |
| `spring-ai-ollama` | `spring-ai-starter-model-ollama` |

命名规则变为：`spring-ai-starter-{类型}-{名称}`，类型包括 `model`、`vector-store` 等。

### 解决

修改 `llm-infrastructure/pom.xml` 中的 artifactId。

---

## 坑 3：QdrantVectorStore 构造方法废弃

### 现象

```java
// 编译报错：构造方法已废弃/移除
new QdrantVectorStore(client, collectionName, embeddingModel, true);
```

### 原因

spring-ai 1.0.1 中 `QdrantVectorStore` 改为 Builder 模式，旧的四参数构造方法被移除。

### 解决

```java
// 旧写法（M4）
return new QdrantVectorStore(client, collectionName, embeddingModel, true);

// 新写法（1.0.1 GA）
return QdrantVectorStore.builder(client, embeddingModel)
        .collectionName(collectionName)
        .initializeSchema(true)
        .build();
```

---

## 坑 4：spring-cloud-starter-bootstrap 版本不匹配

### 现象

升级 Spring Boot 到 3.4.5 后，`spring-cloud-starter-bootstrap 4.1.0` 可能出现兼容性警告或配置加载异常。

### 原因

| spring-cloud-starter-bootstrap | 对应 Spring Cloud | 对应 Spring Boot |
|-------------------------------|-------------------|-----------------|
| 4.1.0 | 2023.0.x | 3.2.x |
| **4.2.0** | **2024.0.x** | **3.4.x** |

### 解决

升级到 `4.2.0`。

---

## 坑 5：YAML 文件中的隐蔽语法错误

### 现象

```
org.yaml.snakeyaml.parser.ParserException: while parsing a block mapping
expected <block end>, but found '<block mapping start>'
```

### 原因

`config/dev/spring-ai.yml` 中 key 前面多了一个点：

```yaml
spring:
  ai:
   .alibaba:    # ← 前导点，YAML 无法解析
```

肉眼很难发现，尤其在缩进不明显的编辑器中。

### 解决

```yaml
spring:
  ai:
    alibaba:    # 去掉点，对齐缩进
```

### 预防

- YAML 文件保存前用 IDE 的 YAML 校验功能检查
- CI 中加入 YAML lint 步骤

---

## 坑 6：如何确认 spring-ai-alibaba 到底依赖哪个 spring-ai 版本

### 问题

官方文档没有明确的版本兼容矩阵，搜索也找不到。

### 解决方法

直接读本地 Maven 缓存中的 POM：

```bash
cat ~/.m2/repository/com/alibaba/cloud/ai/spring-ai-alibaba-core/1.0.0.4/spring-ai-alibaba-core-1.0.0.4.pom
```

在 `<dependencies>` 中可以看到：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-commons</artifactId>
    <version>1.0.1</version>    <!-- 这就是实际依赖的版本 -->
</dependency>
```

**结论**：`spring-ai-alibaba 1.0.0.4` 依赖 `spring-ai 1.0.1`（GA），不是 M4。

---

## 最终版本组合（验证通过）

```xml
<!-- root pom.xml -->
<parent>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.4.5</version>
</parent>

<properties>
    <spring-ai.version>1.0.1</spring-ai.version>
    <spring-ai-alibaba.version>1.0.0.4</spring-ai-alibaba.version>
    <spring-cloud-bootstrap.version>4.2.0</spring-cloud-bootstrap.version>
</properties>
```

### 版本对应关系

```
spring-boot 3.4.5
  └── Spring Framework 6.2.x
        └── spring-ai 1.0.1 (GA)
              └── spring-ai-alibaba 1.0.0.4
                    └── DashScope API (通义千问)
```

---

## 坑 7：手动创建 QdrantClient 缺少 gRPC 依赖

### 现象

```
java: 无法访问io.grpc.ManagedChannel
  找不到io.grpc.ManagedChannel的类文件
```

### 原因

`spring-ai-starter-vector-store-qdrant` 内部通过 auto-configuration 使用 `QdrantClient`，`io.qdrant:client` 和 `io.grpc` 作为传递依赖存在。但当项目中**手动 `new QdrantClient()`**（多 collection 场景需要多个 VectorStore Bean）时，编译器需要直接访问 `io.grpc.ManagedChannel`，而传递依赖的 scope 可能不足以满足编译期可见性。

### 解决

显式引入 Qdrant Java Client：

```xml
<dependency>
    <groupId>io.qdrant</groupId>
    <artifactId>client</artifactId>
    <version>1.13.0</version>
</dependency>
```

### 教训

starter 的传递依赖只保证 starter 内部能用，手动调底层 API 时必须显式声明依赖。

---

## 坑 8：多平台 starter 共存导致 ChatModel Bean 冲突

### 现象

```
Parameter 1 of method chatClientBuilder in ChatClientAutoConfiguration
required a single bean, but 3 were found:
  - dashscopeChatModel
  - ollamaChatModel
  - openAiChatModel
```

### 原因

每个平台的 starter（`spring-ai-alibaba-starter-dashscope`、`spring-ai-starter-model-ollama`、`spring-ai-starter-model-openai`）都会通过 auto-configuration 注册一个 `ChatModel` Bean。而 `ChatClientAutoConfiguration` 要求注入**单一** `ChatModel` 来构建 `ChatClient`，多个就冲突。

### 解决

**1. 只用 Embedding 不用 Chat 的平台，引核心库而非 starter：**

```xml
<!-- 错误：引入 starter，会注册 openAiChatModel -->
<artifactId>spring-ai-starter-model-openai</artifactId>

<!-- 正确：只引核心库，手动构建需要的 Bean -->
<artifactId>spring-ai-openai</artifactId>
```

**2. 确实需要多个 ChatModel 的，排除 ChatClientAutoConfiguration：**

```java
@SpringBootApplication(exclude = {
    ChatClientAutoConfiguration.class
})
```

项目通过自建的 `LLMProviderRouter` 路由多个 `ChatModel`，不依赖 Spring AI 的 `ChatClient`，排除是正确做法。

### 最佳实践总结

| 场景 | 方案 |
|------|------|
| 只用一个平台 | 直接引 starter，不需要额外处理 |
| 多平台共存 | 引多个 starter + 排除 `ChatClientAutoConfiguration` + 自建 Router |
| 只借用某平台的部分能力（如只用 Embedding） | 引核心库（不带 `-starter`），手动构建 Bean |

---

## 坑 9：智谱 API Embedding 路径与 OpenAI 不兼容

### 现象

```
404 - {"status":404,"error":"Not Found","path":"/v1/embeddings"}
```

### 原因

`OpenAiApi` 默认拼接 `/v1/embeddings`，但智谱的实际路径是 `/v4/embeddings`。

### 解决

`OpenAiApi.Builder` 提供了 `embeddingsPath()` 方法可以自定义路径：

```java
OpenAiApi api = OpenAiApi.builder()
        .apiKey(zhipu.getApiKey())
        .baseUrl("https://open.bigmodel.cn/api/paas")
        .embeddingsPath("/v4/embeddings")  // 覆盖默认的 /v1/embeddings
        .build();
```

不要试图通过修改 base-url 来凑路径（如加 `/v4`），会导致 `base-url/v1/embeddings` 变成 `base-url/v4/v1/embeddings` 双重路径。

---

## 经验总结

1. **里程碑版（Mx）不要用于生产**，它的 API 和包路径随时会变
2. **第三方 starter 实际依赖的版本以 POM 为准**，不要相信文档或猜测
3. **升级 Spring Boot 大版本时**，spring-cloud、spring-ai 等生态组件必须同步升级
4. **YAML 语法错误**往往报错位置和实际错误位置不一致，要从报错行往上看
5. **`allow-bean-definition-overriding: true`** 是万恶之源，用它一时爽，后面 debug 火葬场
6. **多平台 starter 共存**必须排除 `ChatClientAutoConfiguration`，自建 Router 管理
7. **只借用某平台部分能力**（如 Embedding）时，引核心库不引 starter，避免注册多余 Bean
8. **第三方 API 兼容 OpenAI 协议不代表路径也兼容**，用 `embeddingsPath()` / `completionsPath()` 自定义
