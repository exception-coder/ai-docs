# 编码摘要文档

> 本文档由完整设计文档精简而来，供 AI 辅助编码时使用，聚焦实现所需的最小必要信息。
> 对应完整文档：`ChatModel统一获取与限速降级-20260409-v1.md`
>
> **职责边界**：设计文档回答"哪些类、什么职责、怎么协作"，本文档回答"每个方法怎么写"。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-09 | 初始版本 |

---

## 1. 核心业务规则

- 规则1：每个 Provider 的 ChatModel 只创建一次（synchronized lazy init），`getChatModel()` 返回同一实例
- 规则2：限速令牌桶使用 Guava `RateLimiter.create(rpm / 60.0)`，将 RPM 转换为每秒许可数
- 规则3：`tryAcquire` 最多等待 5 秒，超时抛 `RateLimitExceededException`
- 规则4：rpm=0 时跳过限速逻辑（Ollama 本地模型场景）
- 规则5：降级顺序由 `llm.fallback-order` 配置决定，主 Provider 优先
- 规则6：平台返回 429（`NonTransientAiException`）时，Provider 需 catch 并包装为 `RateLimitExceededException`
- 规则7：Guava 为已有传递依赖，无需新增 POM 依赖

---

## 2. 接口契约

### 入口接口

无新增 REST 接口，仅内部 Bean 调用变更。

### 内部接口变更

```
LLMProvider#getChatModel(): ChatModel — 新增，返回底层 Spring AI ChatModel 实例
LLMProviderRouter#chatWithFallback(LLMRequest, String): LLMResponse — 新增，带降级的调用
```

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.llm.domain.service.LLMProvider` | 修改 | 新增 `getChatModel()` 默认方法 |
| `com.exceptioncoder.llm.domain.exception.RateLimitExceededException` | 新建 | 限流异常，携带 provider 名称 |
| `com.exceptioncoder.llm.infrastructure.provider.ZhipuProvider` | 修改 | 实现 `getChatModel()`；`chat()`/`chatStream()` 前加限速 |
| `com.exceptioncoder.llm.infrastructure.provider.QwenProvider` | 修改 | 实现 `getChatModel()`；`chat()`/`chatStream()` 前加限速 |
| `com.exceptioncoder.llm.infrastructure.provider.OllamaProvider` | 修改 | 实现 `getChatModel()`；不加限速 |
| `com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter` | 修改 | 新增 `chatWithFallback()` 和 `resolveCandidates()` |
| `com.exceptioncoder.llm.infrastructure.agent.executor.AlibabaAgentExecutor` | 修改 | 注入 Router 替代 LLMConfiguration；删除 3 个 build 方法 |
| `com.exceptioncoder.llm.infrastructure.config.AgentConfiguration` | 修改 | Bean 工厂方法参数从 LLMConfiguration 改为 LLMProviderRouter |
| `com.exceptioncoder.llm.infrastructure.config.LLMConfiguration` | 修改 | 各 Provider Config 新增 `RateLimitConfig` 内部类；顶层新增 `fallbackOrder` |
| `llm-starter/src/main/resources/config/dev/spring-ai.yml` | 修改 | 新增限速和降级配置 |
| `llm-starter/src/main/resources/config/prod/spring-ai.yml` | 修改 | 新增限速和降级配置 |

### 关键方法签名与职责

```
// ====== Domain 层 ======

// LLMProvider 接口新增
com.exceptioncoder.llm.domain.service.LLMProvider#getChatModel(): ChatModel
  — 返回底层 Spring AI ChatModel 实例（已缓存），供 AgentExecutor 等需要直接操作消息列表的场景使用

// RateLimitExceededException 新建
com.exceptioncoder.llm.domain.exception.RateLimitExceededException#RateLimitExceededException(String providerName)
  — 构造方法，message 格式："Provider [{providerName}] 请求频率超限"
com.exceptioncoder.llm.domain.exception.RateLimitExceededException#getProviderName(): String
  — 获取触发限流的 provider 名称

// ====== Infrastructure — Provider 层 ======

// ZhipuProvider 修改
com.exceptioncoder.llm.infrastructure.provider.ZhipuProvider#getChatModel(): ChatModel
  — 调用已有的 getOrCreateChatModel()，直接返回缓存的 OpenAiChatModel 实例
com.exceptioncoder.llm.infrastructure.provider.ZhipuProvider#acquirePermit(): void
  — 私有方法。从 rateLimiter 获取令牌，rpm>0 时 tryAcquire(5, SECONDS)，失败抛 RateLimitExceededException
com.exceptioncoder.llm.infrastructure.provider.ZhipuProvider#initRateLimiter(): void
  — 私有方法。在 getOrCreateChatModel() 中调用，rpm>0 时创建 RateLimiter.create(rpm/60.0)
com.exceptioncoder.llm.infrastructure.provider.ZhipuProvider#chat(LLMRequest): LLMResponse
  — 现有方法，在 model.call() 前插入 acquirePermit() 调用
  — catch NonTransientAiException，若 HTTP 429 则包装为 RateLimitExceededException 重新抛出
com.exceptioncoder.llm.infrastructure.provider.ZhipuProvider#chatStream(LLMRequest): Flux<String>
  — 现有方法，在 model.stream() 前插入 acquirePermit() 调用

// QwenProvider 修改（与 ZhipuProvider 对称）
com.exceptioncoder.llm.infrastructure.provider.QwenProvider#getChatModel(): ChatModel
  — 返回缓存的 DashScopeChatModel 实例
com.exceptioncoder.llm.infrastructure.provider.QwenProvider#acquirePermit(): void
  — 同 ZhipuProvider
com.exceptioncoder.llm.infrastructure.provider.QwenProvider#initRateLimiter(): void
  — 同 ZhipuProvider
com.exceptioncoder.llm.infrastructure.provider.QwenProvider#chat(LLMRequest): LLMResponse
  — 现有方法，插入 acquirePermit()，catch 429 包装
com.exceptioncoder.llm.infrastructure.provider.QwenProvider#chatStream(LLMRequest): Flux<String>
  — 现有方法，插入 acquirePermit()

// OllamaProvider 修改
com.exceptioncoder.llm.infrastructure.provider.OllamaProvider#getChatModel(): ChatModel
  — 返回缓存的 OllamaChatModel 实例
  — 不加限速（本地模型无 RPM 限制）

// ====== Infrastructure — Router 层 ======

com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter#chatWithFallback(LLMRequest request, String preferredProvider): LLMResponse
  — 按 fallbackOrder 解析候选 Provider 列表（preferredProvider 排第一）
  — 遍历候选列表，逐个尝试 provider.chat(request)
  — 捕获 RateLimitExceededException，log.warn 后尝试下一个
  — 所有候选均失败时抛 RuntimeException("所有 Provider 不可用")

com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter#resolveCandidates(String preferredProvider): List<LLMProvider>
  — 私有方法。从 fallbackOrder 配置构建有序 Provider 列表
  — preferredProvider 不为空时移到列表首位
  — 过滤掉 providers 中不存在的配置项

// ====== Infrastructure — AgentExecutor ======

com.exceptioncoder.llm.infrastructure.agent.executor.AlibabaAgentExecutor#AlibabaAgentExecutor(AgentDefinitionRepository, ToolRegistryImpl, ToolExecutor, LLMProviderRouter)
  — 构造方法参数：LLMConfiguration 替换为 LLMProviderRouter
  — 删除字段：llmConfig
  — 新增字段：providerRouter

// 删除以下 3 个方法：
// - buildChatModel(AgentDefinition): ChatModel
// - buildDashScopeChatModel(AgentDefinition): DashScopeChatModel
// - buildZhipuChatModel(AgentDefinition): OpenAiChatModel

com.exceptioncoder.llm.infrastructure.agent.executor.AlibabaAgentExecutor#execute(AgentExecutionRequest): AgentExecutionResult
  — 现有方法，第 76 行 buildChatModel(agent) 替换为：
    LLMProvider provider = providerRouter.route(agent.llmModel() != null ? agent.llmModel() : "default");
    ChatModel chatModel = provider.getChatModel();
  — 其余 ReAct 循环逻辑不变

// ====== Infrastructure — Configuration ======

com.exceptioncoder.llm.infrastructure.config.AgentConfiguration#agentExecutor(...)
  — Bean 工厂方法参数：LLMConfiguration llmConfig 替换为 LLMProviderRouter providerRouter
  — new AlibabaAgentExecutor(..., providerRouter) 替代 new AlibabaAgentExecutor(..., llmConfig)

// LLMConfiguration 新增
com.exceptioncoder.llm.infrastructure.config.LLMConfiguration.RateLimitConfig
  — 内部静态类，字段：int rpm = 0
com.exceptioncoder.llm.infrastructure.config.LLMConfiguration#fallbackOrder: List<String>
  — 顶层新增字段，默认空列表
// 各 Provider Config（ZhipuConfig, AlibabaConfig, OllamaConfig）新增字段：
//   private RateLimitConfig rateLimit = new RateLimitConfig();
```

---

## 4. 数据结构

### 关键表及字段

无数据库变更。

### 关键配置结构

```yaml
# spring-ai.yml 新增配置
llm:
  fallback-order:          # 降级顺序
    - zhipu
    - alibaba
  zhipu:
    rate-limit:
      rpm: 50              # 每分钟最大请求数
  alibaba:
    rate-limit:
      rpm: 50
  ollama:
    rate-limit:
      rpm: 0               # 0 = 不限速
```

---

## 5. 重要约束与边界

- 并发控制：RateLimiter 本身线程安全，无需额外加锁
- 事务范围：无事务，Router 无状态
- 不处理的场景：
  - 多 Key 轮换（后续扩展）
  - TPM（Token Per Minute）限制（后续扩展）
  - 请求队列 / 异步排队（后续扩展）
- domain 层新增 `ChatModel` 依赖：`org.springframework.ai.chat.model.ChatModel`，与已有的 `reactor.core.publisher.Flux` 同属 Spring 生态，可接受

---

## 6. 下游依赖调用

```
// 无外部 Feign/RPC 调用
// 内部调用链：
AlibabaAgentExecutor → LLMProviderRouter#route(String) → LLMProvider#getChatModel()
LLMProviderRouter#chatWithFallback() → LLMProvider#chat(LLMRequest)（逐个尝试）
```

---

## 7. 异常处理要点

- Provider 限速等待超时 → 抛出 `RateLimitExceededException(providerName)`
- 平台返回 429（`NonTransientAiException` 且 message 含 "429"）→ catch 后包装为 `RateLimitExceededException` 抛出
- Router `chatWithFallback` 所有候选均失败 → 抛出 `RuntimeException("所有 Provider 不可用")`
- Router `route` 无匹配 Provider → 不变，抛 `IllegalArgumentException`
