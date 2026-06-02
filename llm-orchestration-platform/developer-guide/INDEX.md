# llm-orchestration-platform / developer-guide 文档索引

## 智能体开发指南
- 文件：`agent-development-guide.md`
- 摘要：在 llm-orchestration-platform 中开发新智能体（Agent Group）的完整流程，从角色定义到工具注册、编排接入、质量门禁
- 大纲：概念模型 / 开发前准备 / 定义角色枚举 / 开发 Tool / 配置 Agent / 编写初始化器 / 实现 Router / 接入 Graph 编排 / 质量门禁 / 可观测性 / 启动顺序与生命周期

## DDD 实践案例：SSE 推送的分层归属决策
- 文件：`ddd-practice-sse-layer-decision.md`
- 摘要：Agent 异步执行功能开发中关于 SSE 推送分层归属的三轮方案演进（API 跨层引用 → Domain 放 SseEmitter → Domain Event + Flux），含决策过程和 DDD 分层实操清单
- 大纲：背景 / 三轮方案演进 / 三方案对比 / 关键认知 / 项目验证 / DDD 分层实操清单

## 最佳实践：用 Stream-Collect 模式解决 LLM 调用超时
- 文件：`best-practice-stream-collect-for-llm-timeout.md`
- 摘要：Agent 执行器调用 LLM API 时因同步阻塞导致 ReadTimeoutException，采用 stream() + collectList() + block() 模式从根本上消除读超时，对 ReAct 循环上层完全透明
- 大纲：背景 / 根因分析 / 解决方案（Stream-Collect 模式）/ 方案对比 / 适用场景判断 / 注意事项 / 总结
