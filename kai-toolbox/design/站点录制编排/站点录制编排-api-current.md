# 站点录制编排 · 接口文档

> 与 [站点录制编排-current.md](./站点录制编排-current.md) 配套。字段级 detail 在本文档维护；流程、风险、设计决策见技术方案文档。
>
> **保留沿用旧接口**：`/api/browser-request/sessions/*`、`/api/browser-request/sessions/{id}/open|save|close|clear` 这一组 session 管理端点本次**不动**，字段沿用旧契约。本文档只覆盖**新增**与**重做**部分。

---

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|--------------|
| v1 | 2026-05-21 | AI | 初版：新增 recording / task / replay 三组端点；删除 saved / vars / pipeline 端点 |

---

## 接口清单

| # | 方法 | 路径 | 用途 |
|---|------|------|------|
| 1 | POST | /api/browser-request/sessions/{id}/recordings | 启动一次录制 |
| 2 | POST | /api/browser-request/recordings/{id}/stop | 停止录制 |
| 3 | GET | /api/browser-request/sessions/{id}/recordings | 列出某 session 的所有录制 |
| 4 | GET | /api/browser-request/recordings/{id} | 获取录制详情（含 call 列表） |
| 5 | DELETE | /api/browser-request/recordings/{id} | 删除录制 |
| 6 | GET | /api/browser-request/recordings/{id}/events | SSE：实时推送录制中的 call |
| 7 | POST | /api/browser-request/tasks | 创建 Task |
| 8 | GET | /api/browser-request/sessions/{id}/tasks | 列出某 session 的所有 task |
| 9 | GET | /api/browser-request/tasks/{id} | 获取 Task 详情 |
| 10 | PUT | /api/browser-request/tasks/{id} | 更新 Task |
| 11 | DELETE | /api/browser-request/tasks/{id} | 删除 Task |
| 12 | POST | /api/browser-request/tasks/{id}/replay | 触发回放 |
| 13 | GET | /api/browser-request/task-runs/{id}/events | SSE：实时推送回放进度 |
| 14 | GET | /api/browser-request/tasks/{id}/runs | 列出某 Task 的历史回放 |
| 15 | GET | /api/browser-request/task-runs/{id} | 获取单次回放详情 |

---

## 1. POST /sessions/{id}/recordings — 启动录制

### 1.1 基本信息

- **方法**：`POST`
- **路径**：`/api/browser-request/sessions/{id}/recordings`
- **用途**：在指定 session 上启动一次新录制；若该 session 已有 active 录制，自动 STOP 旧的再开新的
- **认证**：不需要（内网工具）
- **幂等**：否

### 1.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | sessionId |

#### Body

```json
{
  "name": "拉取文档列表",
  "captureScript": false
}
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `name` | string | 否 | `录制 {YYYY-MM-DD HH:mm}` | 录制名 |
| `captureScript` | boolean | 否 | false | 是否同时录 .js 资源（兼容 JsCapture 旧能力） |

### 1.3 响应

#### 成功（200）

```json
{
  "id": "rec_01HX...",
  "sessionId": "sess_...",
  "name": "拉取文档列表",
  "status": "RECORDING",
  "startedAt": 1716253200000,
  "endedAt": null,
  "callCount": 0,
  "captureScript": false
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | recordingId |
| `status` | enum | RECORDING / STOPPED / ABANDONED / AUTO_STOPPED |
| `startedAt` / `endedAt` | long? | epoch ms |
| `callCount` | int | 已录的 HTTP 调用数 |

#### 错误

| 错误码 | HTTP | 触发场景 |
|--------|------|---------|
| `SESSION_NOT_OPEN` | 409 | session 未 open |
| `SESSION_NOT_FOUND` | 404 | sessionId 不存在 |

---

## 2. POST /recordings/{id}/stop — 停止录制

### 2.1 基本信息

- **方法**：`POST`
- **路径**：`/api/browser-request/recordings/{id}/stop`
- **幂等**：是（重复 STOP 不报错，直接返回当前状态）

### 2.2 请求

无 body。

### 2.3 响应

返回更新后的 `RecordingView`（结构同接口 1.3）。

#### 错误

| 错误码 | HTTP | 触发场景 |
|--------|------|---------|
| `RECORDING_NOT_FOUND` | 404 | recordingId 不存在 |

---

## 3. GET /sessions/{id}/recordings — 列出录制

### 3.1 响应

```json
[
  {
    "id": "rec_01HX...",
    "sessionId": "sess_...",
    "name": "拉取文档列表",
    "status": "STOPPED",
    "startedAt": 1716253200000,
    "endedAt": 1716253260000,
    "callCount": 23
  }
]
```

按 `startedAt DESC`。

---

## 4. GET /recordings/{id} — 录制详情

### 4.1 请求

#### Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `withCalls` | boolean | 否 | false | 是否展开 call 列表 |
| `offset` | int | 否 | 0 | call 分页偏移 |
| `limit` | int | 否 | 50 | call 分页大小（max 200） |

### 4.2 响应

```json
{
  "recording": { /* RecordingView */ },
  "calls": [
    {
      "id": "call_01HX...",
      "recordingId": "rec_01HX...",
      "seq": 1,
      "method": "GET",
      "url": "https://www.yuque.com/api/docs?page=1",
      "resourceType": "XHR",
      "status": 200,
      "elapsedMs": 312,
      "startedAt": 1716253205000,
      "sensitive": false,
      "responseTruncated": false,
      "requestHeaders": { "x-csrf-token": "..." },
      "requestBody": null,
      "responseHeaders": { "content-type": "application/json" },
      "responseBody": "{\"data\":[...]}",
      "initiator": "https://www.yuque.com/dashboard"
    }
  ],
  "callsTotal": 23,
  "callsHasMore": false
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `calls[].sensitive` | boolean | true 时 body 字段为 null（敏感字段过滤） |
| `calls[].responseTruncated` | boolean | true 表示 body 已截断至 256KB |
| `calls[].responseBody` | string? | null 表示 >2MB 未存 |

---

## 5. DELETE /recordings/{id}

级联删 `browser_request_http_call` 同 recording 的所有行。不级联删 Task（Task 的 `recordingId` 设为 null）。

---

## 6. GET /recordings/{id}/events — SSE 实时录制流

见附录 B。

---

## 7. POST /tasks — 创建 Task

### 7.1 基本信息

- **方法**：`POST`
- **路径**：`/api/browser-request/tasks`
- **幂等**：否

### 7.2 请求 Body

```json
{
  "sessionId": "sess_...",
  "recordingId": "rec_...",
  "name": "批量拉文档",
  "steps": [
    {
      "name": "列文档",
      "fromCallId": "call_01HX...",
      "parameterizations": [
        { "field": "query.page", "token": "1", "varName": "page" }
      ]
    },
    {
      "name": "进文档详情",
      "fromCallId": "call_01HY...",
      "parameterizations": [
        { "field": "path", "token": "fxvrxkep", "varName": "docId" }
      ],
      "extracts": [
        { "name": "title", "jsonPath": "$.data.title" }
      ]
    }
  ],
  "params": [
    { "name": "page", "kind": "number", "defaultValue": "1" },
    { "name": "docId", "kind": "string", "defaultValue": "" }
  ],
  "stepIntervalMs": 200,
  "continueOnError": false
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `sessionId` | string | 是 | 所属 session |
| `recordingId` | string? | 否 | 来源录制；adhoc 创建时为 null |
| `name` | string | 是 | task 名（同 session 内不强制唯一） |
| `steps[]` | array | 是 | 步骤列表，至少 1 个 |
| `steps[].fromCallId` | string? | 二选一 | 引用某条 http_call；与 `adhoc` 互斥 |
| `steps[].adhoc` | object? | 二选一 | 手写请求：`{ method, url, headers, body }` |
| `steps[].name` | string | 是 | step 显示名 |
| `steps[].parameterizations[]` | array | 否 | 该 step 的参数化点列表 |
| `steps[].parameterizations[].field` | enum | 是 | `url` / `path` / `query.{key}` / `header.{key}` / `body` |
| `steps[].parameterizations[].token` | string | 是 | 原文中的字符串片段（保存时校验该 token 在原 field 中**恰好出现一次**） |
| `steps[].parameterizations[].varName` | string | 是 | 替换后引用的变量名 |
| `steps[].extracts[]` | array | 否 | 该 step 响应抽取定义 |
| `steps[].extracts[].name` | string | 是 | 变量名（下游 step 可用 `${name}` 引用） |
| `steps[].extracts[].jsonPath` | string | 是 | JSONPath 表达式 |
| `steps[].continueOnError` | boolean | 否 | 该 step 失败是否继续 |
| `params[]` | array | 是 | task 入参定义（运行时填值的字段清单） |
| `params[].name` | string | 是 | 变量名 |
| `params[].kind` | enum | 是 | `string` / `number` / `boolean` |
| `params[].defaultValue` | string | 否 | 默认值（字符串形式，运行时按 kind 转型） |
| `stepIntervalMs` | int | 否（默认 200） | step 间间隔（≥ 0） |
| `continueOnError` | boolean | 否 | task 级失败继续策略（被 step 级覆盖） |

### 7.3 响应

```json
{
  "id": "task_01HZ...",
  "sessionId": "sess_...",
  "recordingId": "rec_...",
  "name": "批量拉文档",
  "stepCount": 2,
  "paramCount": 2,
  "createdAt": 1716253500000,
  "updatedAt": 1716253500000
}
```

#### 错误

| 错误码 | HTTP | 触发场景 |
|--------|------|---------|
| `INVALID_PARAM` | 400 | steps 空 / params 与 token varName 不一致 |
| `PARAMETERIZATION_TOKEN_NOT_FOUND` | 400 | token 不在原文里 |
| `PARAMETERIZATION_TOKEN_AMBIGUOUS` | 400 | token 在原文中出现多次 |
| `CALL_NOT_FOUND` | 404 | `fromCallId` 对应的 http_call 已被删 |

---

## 8. GET /sessions/{id}/tasks

返回 `TaskSummaryView[]`（结构同接口 7.3）。按 `updatedAt DESC`。

---

## 9. GET /tasks/{id} — Task 详情

返回创建时提交的完整结构 + 元数据。`steps[].fromCall` 字段会带上当前 call 的 url/method 等只读副本（即使原 call 已删，task 仍可回放——adhoc 化）。

---

## 10. PUT /tasks/{id}

请求 body 同接口 7.2（不含 `sessionId` / `recordingId`，这两字段创建后不可改）。

---

## 11. DELETE /tasks/{id}

级联删 task_run 历史。

---

## 12. POST /tasks/{id}/replay — 触发回放

### 12.1 基本信息

- **幂等**：否（每次产生新 task_run）

### 12.2 请求 Body

```json
{
  "params": {
    "page": 1,
    "docId": "fxvrxkep8rus8ytb"
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `params` | object | 是 | Map<varName, value>；缺失或类型不符时 step 执行报错 |

### 12.3 响应

```json
{
  "id": "run_02HX...",
  "taskId": "task_01HZ...",
  "status": "RUNNING",
  "startedAt": 1716253600000,
  "stepCount": 2
}
```

立即返回 RUNNING；后续进度走 SSE（接口 13）。

#### 错误

| 错误码 | HTTP | 触发场景 |
|--------|------|---------|
| `SESSION_NOT_OPEN` | 409 | 所属 session 未 open |
| `MISSING_PARAM` | 400 | 缺必填的 param（且无 defaultValue） |
| `TASK_NOT_FOUND` | 404 | taskId 不存在 |

---

## 13. GET /task-runs/{id}/events — SSE 回放进度

见附录 B。

---

## 14. GET /tasks/{id}/runs

返回 `TaskRunSummaryView[]`：按 `startedAt DESC`，默认最近 50 条。

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | runId |
| `taskId` | string | |
| `status` | enum | RUNNING / DONE / FAILED / CANCELLED |
| `startedAt`/`finishedAt` | long? | |
| `okSteps`/`failedSteps` | int | |
| `inputsJson` | string | 当时的 params（截断 1KB） |

---

## 15. GET /task-runs/{id} — 单次回放详情

```json
{
  "id": "run_02HX...",
  "taskId": "task_01HZ...",
  "status": "DONE",
  "startedAt": 1716253600000,
  "finishedAt": 1716253602000,
  "inputs": { "page": 1, "docId": "fxvrxkep..." },
  "stepResults": [
    {
      "stepIndex": 0,
      "stepName": "列文档",
      "status": 200,
      "elapsedMs": 312,
      "finalUrl": "https://www.yuque.com/api/docs?page=1",
      "responseSample": "{\"data\":[...]}",
      "extracted": {},
      "error": null
    },
    {
      "stepIndex": 1,
      "stepName": "进文档详情",
      "status": 200,
      "elapsedMs": 256,
      "extracted": { "title": "示例文档" },
      "error": null
    }
  ],
  "errorMessage": null
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `stepResults[].responseSample` | string? | 响应体截断至 8KB 仅用于查看 |
| `stepResults[].extracted` | object | 本 step 的 extracts 抽取结果 |
| `stepResults[].error` | string? | step 失败时的错误描述（缺变量 / HTTP 异常 / 抽取失败） |

---

## 附录 A：数据库 schema

```sql
-- 保留（不动）：browser_request_session

-- 新增 1：录制元数据
CREATE TABLE IF NOT EXISTS browser_request_recording (
    id              TEXT PRIMARY KEY,
    session_id      TEXT NOT NULL,
    name            TEXT NOT NULL,
    status          TEXT NOT NULL,                    -- RECORDING / STOPPED / ABANDONED / AUTO_STOPPED
    capture_script  INTEGER NOT NULL DEFAULT 0,       -- 是否同时录 .js
    started_at      INTEGER NOT NULL,
    ended_at        INTEGER,
    call_count      INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX IF NOT EXISTS idx_recording_session ON browser_request_recording(session_id, started_at DESC);
CREATE UNIQUE INDEX IF NOT EXISTS idx_recording_active
  ON browser_request_recording(session_id) WHERE status = 'RECORDING';

-- 新增 2：每条 HTTP 调用
CREATE TABLE IF NOT EXISTS browser_request_http_call (
    id                  TEXT PRIMARY KEY,
    recording_id        TEXT NOT NULL,
    seq                 INTEGER NOT NULL,             -- recording 内递增
    method              TEXT NOT NULL,
    url                 TEXT NOT NULL,
    resource_type       TEXT NOT NULL,                -- XHR / FETCH / DOCUMENT / SCRIPT
    request_headers     TEXT,                          -- JSON
    request_body        TEXT,
    status              INTEGER,
    response_headers    TEXT,                          -- JSON
    response_body       TEXT,
    response_truncated  INTEGER NOT NULL DEFAULT 0,
    sensitive           INTEGER NOT NULL DEFAULT 0,
    started_at          INTEGER NOT NULL,
    elapsed_ms          INTEGER,
    initiator           TEXT
);

CREATE INDEX IF NOT EXISTS idx_call_recording ON browser_request_http_call(recording_id, seq ASC);

-- 新增 3：编排好的任务
CREATE TABLE IF NOT EXISTS browser_request_task (
    id            TEXT PRIMARY KEY,
    session_id    TEXT NOT NULL,
    recording_id  TEXT,                                -- 可空（adhoc 创建 / 录制被删）
    name          TEXT NOT NULL,
    steps_json    TEXT NOT NULL,
    params_json   TEXT NOT NULL,
    options_json  TEXT,                                -- { stepIntervalMs, continueOnError }
    created_at    INTEGER NOT NULL,
    updated_at    INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_task_session ON browser_request_task(session_id, updated_at DESC);

-- 新增 4：回放历史
CREATE TABLE IF NOT EXISTS browser_request_task_run (
    id                 TEXT PRIMARY KEY,
    task_id            TEXT NOT NULL,
    status             TEXT NOT NULL,                  -- RUNNING / DONE / FAILED / CANCELLED
    started_at         INTEGER NOT NULL,
    finished_at        INTEGER,
    inputs_json        TEXT,
    step_results_json  TEXT,
    error_message      TEXT
);

CREATE INDEX IF NOT EXISTS idx_run_task ON browser_request_task_run(task_id, started_at DESC);

-- 删除（首次启动迁移）
DROP TABLE IF EXISTS browser_request_saved;
DROP TABLE IF EXISTS browser_request_var;
DROP TABLE IF EXISTS browser_request_pipeline;
DROP TABLE IF EXISTS browser_request_pipeline_run;
```

---

## 附录 B：SSE 接口

### B.1 录制实时流：GET /recordings/{id}/events

- **协议**：`text/event-stream`
- **事件类型**：`call` / `recording-stopped` / `error`

#### 事件 `call`

每录到一条 HTTP 调用，推送一次。

```
event: call
data: {"id":"call_...","seq":17,"method":"GET","url":"...","status":200,"elapsedMs":312,"resourceType":"XHR"}
```

> 推送的是 call 的**轻量视图**（不含完整 body，body 按需通过 GET /recordings/{id}?withCalls 拉）。

#### 事件 `recording-stopped`

```
event: recording-stopped
data: {"status":"STOPPED","reason":"USER_STOP","callCount":23,"endedAt":1716253260000}
```

`reason`：`USER_STOP` / `MAX_DURATION` / `MAX_CALLS` / `SESSION_CLOSED`

### B.2 回放进度流：GET /task-runs/{id}/events

- **协议**：`text/event-stream`
- **事件类型**：`run-started` / `step` / `run-done` / `run-failed` / `error`

#### 事件 `step`

每完成一个 step 推送一次。

```
event: step
data: {"stepIndex":1,"stepName":"进文档详情","status":200,"elapsedMs":256,"extracted":{"title":"..."},"error":null}
```

#### 事件 `run-done` / `run-failed`

```
event: run-done
data: {"status":"DONE","okSteps":2,"failedSteps":0,"finishedAt":1716253602000}
```

```
event: run-failed
data: {"status":"FAILED","abortedAtStep":1,"errorMessage":"上游 step 0 的 extract page_next 取到 null"}
```
