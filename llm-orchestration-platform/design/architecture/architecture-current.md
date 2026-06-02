# 架构设计文档

## 1. 系统概述

LLM Orchestration Platform 是一个企业级的大语言模型编排和管理平台，提供统一的接口来管理和调用多种 LLM 提供商。

## 2. 架构设计

### 2.1 分层架构

采用 DDD-lite 分层架构，清晰分离关注点：

```
┌─────────────────────────────────────┐
│         API Layer (llm-api)         │  REST 接口、DTO 转换
├─────────────────────────────────────┤
│   Application Layer (llm-application)│  用例编排、业务流程
├─────────────────────────────────────┤
│      Domain Layer (llm-domain)      │  核心模型、业务接口
├─────────────────────────────────────┤
│ Infrastructure Layer (llm-infrastructure)│ 外部系统集成
└─────────────────────────────────────┘
```

### 2.2 依赖关系

```
llm-starter
    ├── llm-api
    │   └── llm-application
    │       └── llm-domain
    └── llm-infrastructure
        └── llm-domain (implements)
```

**关键原则**：
- 依赖方向：外层依赖内层
- Infrastructure 实现 Domain 定义的接口
- Domain 层不依赖任何外层

## 3. 核心组件

### 3.1 领域层 (Domain)

**职责**：定义核心业务模型和接口

**核心接口**：
- `LLMProvider`：LLM 提供商抽象
- `ChainExecutor`：Chain 编排器
- `ConversationMemoryRepository`：对话存储

**核心模型**：
- `LLMRequest`：统一的请求模型
- `LLMResponse`：统一的响应模型
- `Message`：对话消息
- `TokenUsage`：Token 统计

### 3.2 应用层 (Application)

**职责**：编排业务用例，协调领域对象

**核心服务**：
- `LLMOrchestrationService`：智能路由，选择合适的 Provider
- `ConversationService`：管理对话历史和上下文

**用例**：
- `ChatUseCase`：完整的对话流程编排

### 3.3 基础设施层 (Infrastructure)

**职责**：实现领域接口，集成外部系统

**Provider 实现**：
- `OpenAIProvider`：基于 Spring AI
- `OllamaProvider`：基于 LangChain4j

**Repository 实现**：
- `InMemoryConversationRepository`：内存存储（可替换为 Redis）

**配置**：
- `LLMConfiguration`：支持动态刷新的配置类

### 3.4 API 层 (API)

**职责**：处理 HTTP 协议，DTO 转换

**控制器**：
- `ChatController`：对话接口

**异常处理**：
- `GlobalExceptionHandler`：统一异常处理

## 4. 核心流程

### 4.1 对话流程

```
用户请求
    ↓
ChatController (协议处理)
    ↓
ChatUseCase (用例编排)
    ↓
ConversationService (获取历史)
    ↓
LLMOrchestrationService (智能路由)
    ↓
OpenAIProvider / OllamaProvider (调用 LLM)
    ↓
ConversationService (保存历史)
    ↓
返回响应
```

### 4.2 Provider 选择策略

1. 如果请求指定了 `provider`，直接使用
2. 否则根据 `model` 名称匹配：
   - `gpt-*` → OpenAIProvider
   - `llama*`, `mistral*` → OllamaProvider
3. 如果都没有，抛出异常

## 5. 扩展点

### 5.1 添加新的 Provider

1. 实现 `LLMProvider` 接口
2. 添加 `@Component` 注解
3. 自动注册到 `LLMOrchestrationService`

### 5.2 添加新的存储

1. 实现 `ConversationMemoryRepository` 接口
2. 替换 `InMemoryConversationRepository`

### 5.3 添加 Chain 编排

1. 定义 `ChainExecutor` 接口
2. 实现具体的 Chain 逻辑
3. 在 Application 层编排多个 LLM 调用

## 6. 配置管理

### 6.1 动态刷新

使用 `@RefreshScope` 支持配置动态刷新：

```java
@RefreshScope
@Configuration
@ConfigurationProperties(prefix = "llm")
public class LLMConfiguration {
    // 配置字段
}
```

**注意**：
- Configuration 类和 Bean 方法都需要 `@RefreshScope`
- 避免 `@RefreshScope` Bean 之间的依赖注入

### 6.2 配置结构

```yaml
llm:
  default-provider: openai
  default-model: gpt-3.5-turbo
  openai:
    api-key: xxx
    model: gpt-3.5-turbo
    temperature: 0.7
  ollama:
    base-url: http://localhost:11434
    model: llama2
```

## 7. 设计原则

### 7.1 SOLID 原则

- **单一职责**：每个类只负责一个功能
- **开闭原则**：通过接口扩展，不修改现有代码
- **里氏替换**：Provider 可以互相替换
- **接口隔离**：接口精简，职责明确
- **依赖倒置**：依赖抽象而非具体实现

### 7.2 DDD 原则

- **领域驱动**：核心业务逻辑在 Domain 层
- **依赖方向**：Infrastructure 实现 Domain 接口
- **聚合根**：LLMRequest/Response 作为聚合根
- **仓储模式**：通过 Repository 访问数据

## 8. 技术选型

| 组件 | 技术 | 原因 |
|------|------|------|
| OpenAI 集成 | Spring AI | 官方支持，集成简单 |
| 本地模型 | LangChain4j | 功能丰富，社区活跃 |
| 配置刷新 | Spring Cloud | 成熟的动态配置方案 |
| 依赖注入 | 构造函数注入 | 提高可测试性 |
| 对话存储 | 内存/Redis | 灵活切换 |

## 9. 后续优化方向

1. **性能优化**
   - 添加缓存层
   - 实现连接池
   - 异步处理

2. **功能增强**
   - Chain 编排引擎
   - Prompt 模板管理
   - Function Calling
   - 流式响应

3. **可观测性**
   - 添加 Metrics
   - 分布式追踪
   - 日志聚合

4. **安全性**
   - API 认证授权
   - 敏感信息加密
   - 限流熔断

