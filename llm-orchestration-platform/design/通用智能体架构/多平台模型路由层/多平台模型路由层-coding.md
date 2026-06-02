# 编码摘要文档

> 本文档由完整设计文档精简而来，供 AI 辅助编码时使用，聚焦实现所需的最小必要信息。
> 对应完整文档：`多平台模型路由层-20260331-v1.md`

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-03-31 | 新增 ModelType 枚举 + LLMProviderRouter，去掉 QwenProvider @Primary |

---

## 1. 核心业务规则

- Router 通过 `List<LLMProvider>` 构造注入，新增 Provider 只需加 `@Component`，Router 无需修改
- 按 `ModelType` 路由：匹配 `type.getProviderName()` == `provider.getProviderName()`
- 按 model 字符串路由：遍历 providers，调用 `provider.supports(model)` 匹配
- 无匹配时抛出 `IllegalArgumentException`，携带 model 名称，不静默 fallback
- `QwenProvider` 去掉 `@Primary`，不再作为默认注入

---

## 2. 接口契约

无新增 REST 接口，仅内部 Bean 调用。

### 关键方法签名（全类名）

```java
com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter#route(ModelType type): LLMProvider
com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter#route(String model): LLMProvider
com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter#getDefault(): LLMProvider
```

---

## 3. 涉及类清单（全类名）

| 全类名 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.llm.domain.model.ModelType` | 新建 | 枚举：ALI、OLLAMA，每个携带 providerName |
| `com.exceptioncoder.llm.infrastructure.provider.LLMProviderRouter` | 新建 | 统一路由，构造注入 List<LLMProvider> |
| `com.exceptioncoder.llm.infrastructure.provider.QwenProvider` | 修改 | 去掉 @Primary 注解 |

---

## 4. 数据结构

无数据库变更。

### ModelType 枚举结构

```java
public enum ModelType {
    ALI("alibaba"),
    OLLAMA("ollama");

    private final String providerName;
}
```

---

## 5. 重要约束与边界

- 无状态设计，Router 线程安全
- 不处理的场景：fallback 降级、负载均衡、成本路由（后续扩展）
- 构造函数注入，禁止 `@Autowired` 字段注入（项目强制规范）

---

## 7. 异常处理要点

- 无匹配 Provider → 抛出 `IllegalArgumentException("No provider supports model: " + model)`
