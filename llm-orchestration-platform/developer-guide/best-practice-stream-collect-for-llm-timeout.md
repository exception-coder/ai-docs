# 最佳实践：用 Stream-Collect 模式解决 LLM 调用超时

> 一次真实的生产问题修复 —— 从 `ReadTimeoutException` 到 Stream-Collect，
> 记录 Agent 执行器调用 LLM API 时因响应慢导致读超时的根因分析与解决方案。

---

## 1. 背景

### 1.1 问题现象

Agent 执行器在调用智谱 GLM API 时，频繁出现 `ReadTimeoutException`：

```
WARN  --- [pool-4-thread-1] o.s.ai.retry.RetryUtils : Retry error. Retry count:1

org.springframework.web.client.ResourceAccessException:
  I/O error on POST request for
  "https://open.bigmodel.cn/api/paas/v4/chat/completions": null
Caused by: io.netty.handler.timeout.ReadTimeoutException: null
```

触发重试后仍然超时，最终导致 Agent 任务执行失败。

### 1.2 调用链路

```
AlibabaAgentExecutor.execute()
  → callWithRetry(chatModel, messages, iteration)
    → chatModel.call(prompt)          // 同步阻塞，等待完整响应
    → response.getResults().get(0)... // 解析完整响应
  → parseAndExecuteTool(content, ...)
  → 循环直到最终回答
```

### 1.3 技术栈

| 组件 | 版本 |
|------|------|
| Spring AI | 1.0.1 |
| Spring Boot | 3.4.5 |
| HTTP 客户端 | Reactor Netty（Spring AI 默认） |
| LLM API | 智谱 GLM（OpenAI 兼容协议） |

---

## 2. 根因分析

### 2.1 同步调用的超时机制

`chatModel.call(prompt)` 是同步阻塞调用。底层 Reactor Netty 的 `ReadTimeout` 计时器在发出请求后开始倒计时，如果在超时窗口内**没有收到任何数据**，就会抛出 `ReadTimeoutException`。

```
客户端                         LLM API
  |--- POST /chat/completions --->|
  |                                |  ← 模型推理中（可能 30s~120s+）
  |         （无数据返回）           |
  |   ReadTimeoutException!        |
  |<------- 连接断开 -------------|
```

关键点：**同步模式下，整个推理过程中客户端收不到任何数据**。LLM 生成长文本或遇到复杂推理任务时，服务端处理时间很容易超过 Netty 默认的读超时（通常 30s）。

### 2.2 为什么简单加大超时不是好方案

| 方案 | 问题 |
|------|------|
| 加大 ReadTimeout 到 180s | 连接池长时间被占用；如果真的出问题（网络断开），要等 180s 才能感知 |
| 配置 Spring HTTP 全局超时 | 影响所有 HTTP 调用，不仅仅是 LLM |
| Spring AI 自身未暴露超时参数 | `OpenAiApi.Builder` 没有 timeout 字段，需要自定义 `RestClient` |

### 2.3 流式调用的天然优势

LLM API 的流式模式（SSE）在生成第一个 token 后就开始推送数据，**每个 token 都是一次数据到达**：

```
客户端                         LLM API
  |--- POST /chat/completions --->|
  |      (stream: true)           |
  |<-- data: {"content":"你"} ----|  ← 首个 token（~2s）
  |<-- data: {"content":"好"} ----|  ← 后续 token 持续到达
  |<-- data: {"content":"，"} ----|
  |         ...                    |
  |<-- data: [DONE] --------------|  ← 生成完毕
```

每次收到 token，ReadTimeout 计时器都会**重置**。只要 LLM 在持续生成，连接就不会超时。

---

## 3. 解决方案：Stream-Collect 模式

### 3.1 核心思路

**用流式调用替代同步调用，但在内部收集所有 token 拼接为完整响应**。对上层（ReAct 循环）完全透明——返回值仍是 `String`，但底层已从"等一整块"变为"逐片收集"。

```
                    Stream-Collect 模式
                    ==================
LLM API ──stream──> [token₁] [token₂] [token₃] ... [DONE]
                         │        │        │           │
                         └────────┴────────┴───────────┘
                                     │
                                collectList()
                                     │
                               String.join("")
                                     │
                              完整响应字符串
                                     ↓
                         ReAct 循环正常解析
```

### 3.2 改造前后对比

**改造前（同步调用）：**

```java
private String callWithRetry(ChatModel chatModel,
                              List<Message> messages, int iteration) {
    int maxRetries = 2;
    Exception lastException = null;

    for (int retry = 0; retry <= maxRetries; retry++) {
        try {
            Prompt prompt = new Prompt(messages);
            ChatResponse response = chatModel.call(prompt);  // 同步阻塞
            return response.getResults().get(0).getOutput().getText();
        } catch (Exception e) {
            lastException = e;
            if (retry < maxRetries) {
                log.warn("LLM 调用失败，重试 {}/{}", retry + 1, maxRetries, e);
            }
        }
    }
    throw new RuntimeException("LLM 调用失败", lastException);
}
```

**改造后（Stream-Collect）：**

```java
private String callWithRetry(ChatModel chatModel,
                              List<Message> messages, int iteration) {
    int maxRetries = 2;
    Exception lastException = null;

    for (int retry = 0; retry <= maxRetries; retry++) {
        try {
            Prompt prompt = new Prompt(messages);
            // 流式调用：token 逐个返回，避免长时间无数据导致读超时
            String content = chatModel.stream(prompt)
                    .map(response -> {
                        var results = response.getResults();
                        if (results == null || results.isEmpty()) return "";
                        String text = results.get(0).getOutput().getText();
                        return text != null ? text : "";
                    })
                    .collectList()
                    .map(chunks -> String.join("", chunks))
                    .block();
            return content;
        } catch (Exception e) {
            lastException = e;
            if (retry < maxRetries) {
                log.warn("LLM 流式调用失败，重试 {}/{}", retry + 1, maxRetries, e);
            }
        }
    }
    throw new RuntimeException("LLM 调用失败", lastException);
}
```

### 3.3 关键 API 说明

| 方法 | 作用 |
|------|------|
| `chatModel.stream(prompt)` | 返回 `Flux<ChatResponse>`，每个元素包含一个增量 token |
| `.map(response -> ...)` | 从每个 ChatResponse 中提取文本片段，处理 null 边界 |
| `.collectList()` | 将所有片段收集到 `List<String>` 中，返回 `Mono<List<String>>` |
| `.map(chunks -> String.join("", chunks))` | 将片段列表拼接为完整字符串 |
| `.block()` | 阻塞等待 Mono 完成，返回最终 String |

### 3.4 executeStream 的同步改造

同样的思路应用于 `executeStream()` 方法，使其成为真正的流式输出：

```java
@Override
public Flux<String> executeStream(AgentExecutionRequest request) {
    return Flux.create(sink -> {
        try {
            // ... 初始化 Agent、ChatModel、Messages ...

            for (int iteration = 1; iteration <= maxIterations && !finished; iteration++) {
                StringBuilder contentBuilder = new StringBuilder();

                // 流式收集本轮响应
                chatModel.stream(new Prompt(messages))
                        .doOnNext(response -> {
                            String text = response.getResults().get(0).getOutput().getText();
                            if (text != null) {
                                contentBuilder.append(text);
                            }
                        })
                        .blockLast();

                String content = contentBuilder.toString();
                ToolCallResult toolCallResult = parseAndExecuteTool(content, ...);

                if (toolCallResult != null) {
                    // 工具调用迭代：追加 Observation，继续循环
                    messages.add(new AssistantMessage(content));
                    messages.add(new UserMessage("观察结果: " + toolCallResult.observation));
                } else {
                    // 最终回答：发送给调用方
                    sink.next(content);
                    finished = true;
                }
            }
            sink.complete();
        } catch (Exception e) {
            sink.error(e);
        }
    });
}
```

---

## 4. 方案对比

| 维度 | 同步 call() | 加大超时 | Stream-Collect（本方案） |
|------|------------|---------|------------------------|
| 超时风险 | 高：无数据期间计时 | 中：治标不治本 | 低：token 持续到达，计时器不断重置 |
| 连接池影响 | 正常 | 长连接占用加大 | 正常（流式连接及时释放） |
| 改造成本 | 无 | 低 | 低：仅改 callWithRetry 内部 |
| 对上层透明 | 基准 | 完全透明 | 完全透明（返回值类型不变） |
| 错误感知 | 超时后才知道 | 超时后才知道 | 快速失败：连接/鉴权错误在首个 chunk 就暴露 |
| 适用框架 | Spring AI call() | 需自定义 RestClient | Spring AI stream()（原生支持） |

---

## 5. 适用场景判断

### 5.1 推荐使用 Stream-Collect 的场景

- LLM 推理时间不可控（长文本生成、复杂推理任务）
- Agent 执行器的 ReAct 循环（每轮都需要完整响应来判断工具调用）
- 后端 HTTP 超时配置不可控或不方便修改
- 需要快速失败（连接错误、鉴权失败在首个 chunk 就能感知）

### 5.2 仍可使用同步 call() 的场景

- 调用延迟可控且稳定（如 Embedding 接口，通常 <5s）
- 已配置足够的 ReadTimeout 且不影响其他 HTTP 调用
- 不需要流式能力的简单问答场景

---

## 6. 注意事项

### 6.1 null 安全

流式响应的每个 chunk 可能出现 `results` 为空或 `text` 为 null 的情况（如心跳包、元数据事件），必须做防御处理：

```java
.map(response -> {
    var results = response.getResults();
    if (results == null || results.isEmpty()) return "";
    String text = results.get(0).getOutput().getText();
    return text != null ? text : "";
})
```

### 6.2 block() 的线程安全

`block()` 在 Reactor 的非阻塞线程（如 Netty event loop）上调用会抛 `IllegalStateException`。本项目的 Agent 执行器运行在 `ThreadPoolExecutor` 的工作线程上，不受此限制。如果未来改为 WebFlux 响应式栈，需要改用 `subscribeOn(Schedulers.boundedElastic())` 包裹。

### 6.3 重试机制保持不变

Stream-Collect 模式下重试逻辑与同步模式完全一致——整个流失败时才触发重试，不会出现"收了一半重试"的问题。`collectList().block()` 要么返回完整结果，要么整体抛异常。

---

## 7. 总结

| 要点 | 说明 |
|------|------|
| 根因 | 同步 `call()` 在 LLM 推理期间无数据到达，触发 Netty ReadTimeout |
| 方案 | `stream()` + `collectList()` + `block()` —— 流式收集，同步返回 |
| 原理 | 流式模式下每个 token 都重置超时计时器，从根本上避免读超时 |
| 改造范围 | 仅改 `callWithRetry` 方法内部，对 ReAct 循环上层完全透明 |
| 适用范围 | 所有响应时间不可控的 LLM 调用场景 |
