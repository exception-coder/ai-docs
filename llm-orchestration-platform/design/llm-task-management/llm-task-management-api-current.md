# 接口文档：LLM 任务管理统一接口

> 字段级契约维护在本文档；设计决策 / 流程 / 风险见配套 `llm-task-management-current.md`。
> 业务方提交任务的接口（`POST /api/v1/agents/{id}/execute`、`POST /api/v1/dev-plan/submit` 等）维护在各业务方文档，本接口仅承载平台级统一管理能力。

---

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|--------------|
| v1 | 2026-05-25 | zhangkai | 初始版本，5 个统一管理接口 + SSE |

---

## 接口清单

| # | 方法 | 路径 | 用途 |
|---|------|------|------|
| 1 | GET  | /api/v1/tasks | 分页列出任务（按用户/类型/状态过滤） |
| 2 | GET  | /api/v1/tasks/{taskId} | 查询单个任务状态 |
| 3 | GET  | /api/v1/tasks/{taskId}/events | 查询任务历史事件（断线重连前可批量拉取） |
| 4 | GET  | /api/v1/tasks/{taskId}/stream | SSE 订阅任务事件流（支持 Last-Event-ID 断线重连） |
| 5 | POST | /api/v1/tasks/{taskId}/cancel | 请求取消任务（协作式） |

---

## 1. 分页列出任务

### 1.1 基本信息

- **方法**：`GET`
- **路径**：`/api/v1/tasks`
- **用途**：分页列出当前用户的任务，可按 task_type / status 过滤
- **认证**：需要（取当前用户 user_id）
- **幂等**：是（GET）

### 1.2 请求

#### Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `taskType` | string | 否 | - | `AGENT/DEVPLAN/CHAT/RESUME`，不填则全部 |
| `status` | string | 否 | - | 状态值，多状态用逗号分隔 |
| `userId` | string | 否 | 当前用户 | 管理员可指定，普通用户忽略 |
| `page` | integer | 否 | 0 | 0-based |
| `size` | integer | 否 | 20 | 上限 100 |

### 1.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": {
    "page": 0,
    "size": 20,
    "total": 137,
    "items": [
      {
        "taskId": "ag-20260525-001",
        "taskType": "AGENT",
        "bizRef": "secretary-default",
        "status": "COMPLETED",
        "progress": 100,
        "currentStep": null,
        "userId": "u-001",
        "retryCount": 0,
        "createdAt": "2026-05-25T10:12:33",
        "startedAt": "2026-05-25T10:12:33",
        "finishedAt": "2026-05-25T10:13:05"
      }
    ]
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `items[].taskId` | string | 业务可读 ID |
| `items[].taskType` | string | AGENT / DEVPLAN / CHAT / RESUME / ... |
| `items[].bizRef` | string | 业务关联键（agent_id / project_path 等） |
| `items[].status` | string | 枚举见 R2 |
| `items[].progress` | integer | 0~100 |
| `items[].currentStep` | string | 当前步骤描述（运行时） |
| `items[].retryCount` | integer | 已重试次数 |

### 1.4 示例

```http
GET /api/v1/tasks?taskType=AGENT&status=RUNNING,COMPLETED&page=0&size=20 HTTP/1.1
```

---

## 2. 查询单个任务

### 2.1 基本信息

- **方法**：`GET`
- **路径**：`/api/v1/tasks/{taskId}`
- **用途**：查询任务最新状态
- **认证**：需要

### 2.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务业务 ID |

### 2.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": {
    "taskId": "ag-20260525-001",
    "taskType": "AGENT",
    "bizRef": "secretary-default",
    "status": "RUNNING",
    "progress": 40,
    "currentStep": "执行工具 schedule_list",
    "retryCount": 0,
    "maxRetries": 0,
    "timeoutSeconds": 600,
    "lastEventSeq": 7,
    "errorMessage": null,
    "resultJson": null,
    "traceId": "trace-7f8a...",
    "createdAt": "2026-05-25T10:12:33",
    "startedAt": "2026-05-25T10:12:33",
    "finishedAt": null,
    "elapsedMs": 12450
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `lastEventSeq` | integer | 已发布的最大事件序号，前端可据此发起 events / stream 请求 |
| `resultJson` | string | 终态时填充，JSON 字符串 |
| `elapsedMs` | integer | 实时计算（startedAt → finishedAt or now） |

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `TASK_NOT_FOUND` | 404 | taskId 不存在 |
| `FORBIDDEN` | 403 | 非任务所有者访问 |

---

## 3. 查询任务历史事件

### 3.1 基本信息

- **方法**：`GET`
- **路径**：`/api/v1/tasks/{taskId}/events`
- **用途**：批量拉取任务事件历史；前端可用于断线重连前先拉历史、或终态任务的执行回放
- **认证**：需要

### 3.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务业务 ID |

#### Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `afterSeq` | integer | 否 | 0 | 仅返回 `event_seq > afterSeq` 的事件 |
| `limit` | integer | 否 | 500 | 上限 1000 |

### 3.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": {
    "taskId": "ag-20260525-001",
    "events": [
      {
        "seq": 1,
        "type": "STARTED",
        "data": "{\"agentId\":\"secretary-default\"}",
        "createdAt": "2026-05-25T10:12:33.421"
      },
      {
        "seq": 2,
        "type": "ITERATION",
        "data": "{\"iteration\":1,\"thought\":\"...\"}",
        "createdAt": "2026-05-25T10:12:35.108"
      }
    ]
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `events[].seq` | integer | 单调递增序号 |
| `events[].type` | string | `STARTED/ITERATION/TOOL_CALL/TOOL_RESULT/PROGRESS/COMPLETE/ERROR/TIMEOUT/CANCELLED/INTERRUPTED` |
| `events[].data` | string | JSON 字符串载荷，由发布方决定结构 |

---

## 4. SSE 订阅任务事件流

### 4.1 基本信息

- **方法**：`GET`
- **路径**：`/api/v1/tasks/{taskId}/stream`
- **用途**：实时订阅任务事件流；支持 `Last-Event-ID` 头实现断线重连
- **认证**：需要
- **协议**：Server-Sent Events（`Content-Type: text/event-stream`）

### 4.2 请求

#### Headers

| Name | 必填 | 说明 |
|------|------|------|
| `Last-Event-ID` | 否 | 客户端已收到的最大 seq；服务端从 seq+1 开始回放 + 切换热流 |
| `Accept` | 是 | `text/event-stream` |

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务业务 ID |

### 4.3 响应

#### 成功（200）

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

event: started
id: 1
data: {"taskId":"ag-20260525-001","agentId":"secretary-default"}

event: iteration
id: 2
data: {"iteration":1,"thought":"用户想看今天的日程..."}

event: tool_call
id: 3
data: {"toolId":"schedule_list","inputJson":"{}"}

event: tool_result
id: 4
data: {"toolId":"schedule_list","output":"[...]","durationMs":42}

event: progress
id: 5
data: {"percent":60,"step":"已完成 1 个工具调用"}

event: complete
id: 6
data: {"finalOutput":"...","iterations":2,"elapsedMs":3400}

event: done
data: [DONE]
```

#### 事件类型

| 事件 (`event:`) | data 关键字段 | 触发 |
|----|----|----|
| `started` | `agentId / bizRef / taskType` | 任务进入 RUNNING |
| `iteration` | `iteration / thought` | LLM 一轮推理完成 |
| `tool_call` | `toolId / inputJson` | 工具调用前 |
| `tool_result` | `toolId / output / durationMs / success` | 工具调用后 |
| `progress` | `percent / step` | 业务方主动上报进度 |
| `complete` | `finalOutput / iterations / elapsedMs` | 任务正常完成 |
| `error` | `errorMessage / stackHint` | 任务异常 |
| `timeout` | `timeoutSeconds` | 超时 |
| `cancelled` | `reason` | 收到取消请求 |
| `interrupted` | `reason` | 进程重启恢复时回放 |
| `done` | `[DONE]` 文本 | 流结束哨兵 |

**id 字段说明**：每条事件的 `id:` 即 `event_seq`，前端按浏览器规范自动作为下次重连的 `Last-Event-ID`。

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `TASK_NOT_FOUND` | 404 | 流建立时即返回 404 |

#### 终态任务的处理

- 任务已是终态（COMPLETED/FAILED/CANCELLED/TIMED_OUT）时，接入 SSE 会回放全部历史事件 → 发送 `done` → 关闭流
- 任务被 `INTERRUPTED` 时，回放 INTERRUPTED 事件 + done

---

## 5. 请求取消任务

### 5.1 基本信息

- **方法**：`POST`
- **路径**：`/api/v1/tasks/{taskId}/cancel`
- **用途**：标记任务为取消请求中，执行线程在下一个 checkpoint 终止
- **认证**：需要
- **幂等**：是（重复 cancel 视为已请求）

### 5.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `taskId` | string | 是 | 任务业务 ID |

#### Body

```json
{
  "reason": "用户主动取消"
}
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `reason` | string | 否 | "user requested" | 取消原因，写入 CANCELLED 事件载荷 |

### 5.3 响应

#### 成功（202）

```json
{
  "code": 0,
  "data": {
    "taskId": "ag-20260525-001",
    "cancellationRequested": true,
    "currentStatus": "RUNNING"
  }
}
```

**说明**：返回 202 表示"已收到取消请求"，但任务实际状态由执行线程在下一 checkpoint 决定。前端应继续监听 SSE 等待 `cancelled` 事件确认。

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `TASK_NOT_FOUND` | 404 | taskId 不存在 |
| `TASK_ALREADY_TERMINAL` | 409 | 任务已是终态（COMPLETED/FAILED/CANCELLED/TIMED_OUT/INTERRUPTED），无法取消 |
| `FORBIDDEN` | 403 | 非任务所有者 |
