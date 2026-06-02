# 开发日志 2026-W14（2026-03-30 ~ 2026-03-30）

## 2026-03-30

### 任务描述
修复 Spring AI 版本兼容性问题，确保 `mvn clean install -DskipTests` 全量构建通过。

### 变更文件

- `llm-infrastructure/.../provider/OllamaProvider.java`：`Usage.getGenerationTokens()` → `getCompletionTokens()`
- `llm-infrastructure/.../provider/QwenProvider.java`：同上
- `llm-infrastructure/.../repository/DocSearchRepositoryImpl.java`：`Document.getContent()` → `getText()`（Spring AI 1.0.1 API）
- `llm-application/.../service/DocViewerService.java`：`File.list()` → `listFiles()`，修复 `String[]` 赋值 `File[]` 类型错误
- `llm-starter/.../config/DelegatingEmbeddingModel.java`：移除 `OpenAiEmbeddingModel` 依赖，改为仅接受 `OllamaEmbeddingModel`
- `llm-starter/.../config/SpringAIConfiguration.java`：同上，移除 `spring-ai-openai` 相关 import

### 关键设计决策

- Spring AI 1.0.1 的 `Usage` 接口方法为 `getCompletionTokens()`（非 `getGenerationTokens()`），返回 `Integer`（非 `Long`）
- Spring AI 1.0.1 的 `Document` 类使用 `getText()` 而非 `getContent()`
- 项目实际使用 DashScope/Ollama 作为 Embedding 提供商，无需 `spring-ai-openai` 依赖
