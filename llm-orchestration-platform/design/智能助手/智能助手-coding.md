# 编码摘要 — 智能助手

## 变更记录
- v1: 初始版本，秘书 Agent（ReAct）+ 日程/待办/记忆工具 + 独立前端页面

---

## 涉及类清单

| 全类名 | 所属模块 | 职责 |
|---|---|---|
| `com.exceptioncoder.llm.api.controller.SecretaryController` | llm-api | HTTP 入口，SSE 流式编排 |
| `com.exceptioncoder.llm.application.service.SecretaryService` | llm-application | 组装 systemPrompt + 记忆，调用执行器 |
| `com.exceptioncoder.llm.domain.model.SecretaryMemory` | llm-domain | 记忆领域模型 |
| `com.exceptioncoder.llm.domain.model.SecretarySchedule` | llm-domain | 日程领域模型 |
| `com.exceptioncoder.llm.domain.model.SecretaryTodo` | llm-domain | 待办领域模型 |
| `com.exceptioncoder.llm.domain.repository.SecretaryMemoryRepository` | llm-domain | 记忆仓储接口 |
| `com.exceptioncoder.llm.domain.repository.SecretaryScheduleRepository` | llm-domain | 日程仓储接口 |
| `com.exceptioncoder.llm.domain.repository.SecretaryTodoRepository` | llm-domain | 待办仓储接口 |
| `com.exceptioncoder.llm.infrastructure.entity.SecretaryMemoryEntity` | llm-infrastructure | 记忆 JPA 实体 |
| `com.exceptioncoder.llm.infrastructure.repository.SecretaryMemoryRepositoryImpl` | llm-infrastructure | 记忆仓储实现 |
| `com.exceptioncoder.llm.infrastructure.repository.SecretaryScheduleRepositoryImpl` | llm-infrastructure | 日程仓储实现 |
| `com.exceptioncoder.llm.infrastructure.repository.SecretaryTodoRepositoryImpl` | llm-infrastructure | 待办仓储实现 |
| `com.exceptioncoder.llm.infrastructure.provider.Tool` | llm-infrastructure | 工具注解（@Tool + @ToolParam） |
| `com.exceptioncoder.llm.infrastructure.config.ScheduleTool` | llm-infrastructure | 日程管理 @Tool Bean |
| `com.exceptioncoder.llm.infrastructure.config.TodoTool` | llm-infrastructure | 待办管理 @Tool Bean |
| `com.exceptioncoder.llm.infrastructure.config.SecretaryConfiguration` | llm-infrastructure | 工具插拔 @ConditionalOnProperty |
| `com.exceptioncoder.llm.infrastructure.agent.SecretaryAgentInitializer` | llm-infrastructure | 启动时注册 secretary-default Agent |
| `com.exceptioncoder.llm.infrastructure.agent.ToolScanner` | llm-infrastructure | 扫描 @Tool Bean 注册到 Registry |

---

## 接口设计

| 方法 | 路径 | 方法签名 |
|---|---|---|
| POST | `/api/v1/secretary/chat` | `SecretaryController.chat(ChatRequest): ResponseEntity<AgentExecutionResult>` |
| POST | `/api/v1/secretary/chat/stream` | `SecretaryController.chatStream(ChatRequest): Flux<ServerSentEvent<String>>` |
| GET | `/api/v1/secretary/memory` | `SecretaryController.getMemory(): ResponseEntity<List<SecretaryMemory>>` |
| POST | `/api/v1/secretary/memory` | `SecretaryController.saveMemory(SecretaryMemory): ResponseEntity<Void>` |
| DELETE | `/api/v1/secretary/memory` | `SecretaryController.clearMemory(): ResponseEntity<Void>` |
| GET | `/api/v1/secretary/tools` | `SecretaryController.getTools(): ResponseEntity<List<ToolDefinition>>` |

---

## 核心业务规则

| 规则 | 说明 |
|---|---|
| 日程时间格式 | 输入/输出统一使用 `yyyy-MM-dd HH:mm` |
| 优先级枚举 | 仅接受 `LOW/MEDIUM/HIGH/URGENT`，其他值默认 `MEDIUM` |
| 记忆类型枚举 | `PREFERENCE/SUMMARY/PROFILE`，LLM 决定插入类型 |
| 工具默认启用 | `@ConditionalOnProperty` 配置 `matchIfMissing = true`，不配置则默认启用 |
| Agent ID 固定 | `secretary-default`，初始化器确保幂等 |
| Session ID | 前端每次新建页面生成 `session-{timestamp}` |

---

## 数据库设计

### 表清单

```sql
CREATE TABLE secretary_memory (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL,           -- PREFERENCE / SUMMARY / PROFILE
    content TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_secretary_memory_user (user_id)
);

CREATE TABLE secretary_schedule (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    title VARCHAR(200) NOT NULL,
    description VARCHAR(500),
    start_time DATETIME,
    end_time DATETIME,
    reminder TINYINT(1) NOT NULL DEFAULT 0,
    done TINYINT(1) NOT NULL DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_secretary_schedule_user (user_id),
    INDEX idx_secretary_schedule_time (start_time)
);

CREATE TABLE secretary_todo (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(100) NOT NULL,
    title VARCHAR(200) NOT NULL,
    priority VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',  -- LOW / MEDIUM / HIGH / URGENT
    due_date DATE,
    done TINYINT(1) NOT NULL DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_secretary_todo_user (user_id)
);
```

---

## 事务与并发

- 日程/待办操作均为单表写，无需多表事务
- `SecretaryAgentInitializer` 使用 `existsById` 防止重复初始化（幂等设计）
- 并发安全：`ConcurrentHashMap` 在 `ToolRegistryImpl` 内部管理，无并发写冲突
