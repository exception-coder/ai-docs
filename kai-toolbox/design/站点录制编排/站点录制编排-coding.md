# 站点录制编排 · 编码摘要

> 配套：[站点录制编排-current.md](./站点录制编排-current.md) · [API 文档](./站点录制编排-api-current.md)
>
> **职责边界**：本文件只列方法签名、关键实现要点、约束。设计决策见 current.md，字段级契约见 api-current.md。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-05-21 | 初版 |

---

## 1. 核心业务规则（编码不得违反）

- 录制时 `resourceType` 过滤集合默认 `{XHR, FETCH, DOCUMENT}`；`captureScript=true` 时再加 `SCRIPT`
- URL/请求体字段名含 `password` / `pwd` / `token` / `secret` 关键词 → `sensitive=1` + body 字段写 `null`
- response body：≤256KB 全存；256KB~2MB 截断到 256KB + `responseTruncated=1`；>2MB 不存 body
- 单 session 最多一条 `status=RECORDING` 的录制（DB UNIQUE 部分索引保障 + 应用层 START 前先 STOP 旧的）
- 录制硬上限：60 min（来自 `props.recordingMaxDurationMs`）/ 5000 call（`recordingMaxCalls`）；触顶 → `status=AUTO_STOPPED` + SSE 推 `recording-stopped { reason: MAX_DURATION/MAX_CALLS }`
- task.steps 严格按数组下标顺序执行；step 之间 `stepIntervalMs` 节流（默认 200ms）
- `Parameterization.token` 在原 field 中必须**恰好出现一次**，否则保存时 400
- step 间引用：先查同 task_run 已 extract 出的 outputs，再查用户填入的 params；都没有 → step 失败
- step 失败默认 stop-on-error；step 级 `continueOnError=true` 才继续；task 级标志仅作为 step 级缺省值
- 后端启动时 `UPDATE recording SET status='ABANDONED' WHERE status='RECORDING'`（无法续录）
- 删 recording 不级联删 task；task.recording_id 改为 NULL（task 仍可回放，因 steps 已副本化）
- 删 task **级联删** task_run 历史

---

## 2. 接口入口指针

| 接口 | 实现类 #方法 |
|------|-------------|
| `POST /api/browser-request/sessions/{id}/recordings` | `BrowserRequestController#startRecording` → `RecordingService#start` |
| `POST /api/browser-request/recordings/{id}/stop` | `BrowserRequestController#stopRecording` → `RecordingService#stop` |
| `GET /api/browser-request/sessions/{id}/recordings` | `BrowserRequestController#listRecordings` → `RecordingService#listBySession` |
| `GET /api/browser-request/recordings/{id}` | `BrowserRequestController#getRecording` → `RecordingService#detail` |
| `DELETE /api/browser-request/recordings/{id}` | `BrowserRequestController#deleteRecording` → `RecordingService#delete` |
| `GET /api/browser-request/recordings/{id}/events` (SSE) | `BrowserRequestController#recordingEvents` → 用 `SseEmitterRegistry` 注册订阅 key |
| `POST /api/browser-request/tasks` | `BrowserRequestController#createTask` → `TaskService#create` |
| `GET /api/browser-request/sessions/{id}/tasks` | `BrowserRequestController#listTasks` → `TaskService#listBySession` |
| `GET /api/browser-request/tasks/{id}` | `BrowserRequestController#getTask` → `TaskService#detail` |
| `PUT /api/browser-request/tasks/{id}` | `BrowserRequestController#updateTask` → `TaskService#update` |
| `DELETE /api/browser-request/tasks/{id}` | `BrowserRequestController#deleteTask` → `TaskService#delete` |
| `POST /api/browser-request/tasks/{id}/replay` | `BrowserRequestController#replay` → `ReplayExecutor#replay` |
| `GET /api/browser-request/task-runs/{id}/events` (SSE) | `BrowserRequestController#replayEvents` → `SseEmitterRegistry` |
| `GET /api/browser-request/tasks/{id}/runs` | `BrowserRequestController#listRuns` → `TaskService#listRuns` |
| `GET /api/browser-request/task-runs/{id}` | `BrowserRequestController#getRun` → `TaskService#runDetail` |

---

## 3. 涉及类清单（全路径）

### 后端 — Java（`tools/tool-browser-request/src/main/java/com/exceptioncoder/toolbox/browserrequest/`）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `...api.BrowserRequestController` | 修改 | 删 saved/var/pipeline 端点；新增 recording/task/replay/SSE 共 15 个端点 |
| `...api.dto.ExecuteRequestBody` | 删除 | |
| `...api.dto.ExtractToSavedRequest` | 删除 | |
| `...api.dto.PipelineDtos` | 删除 | |
| `...api.dto.SaveRequestBody` | 删除 | |
| `...api.dto.UpsertVarRequest` | 删除 | |
| `...api.dto.StartRecordingRequest` | 新建 | record `{ String name, Boolean captureScript }` |
| `...api.dto.CreateTaskRequest` | 新建 | record（结构对照 api 文档 7.2） |
| `...api.dto.UpdateTaskRequest` | 新建 | record（不含 sessionId / recordingId） |
| `...api.dto.ReplayRequest` | 新建 | record `{ Map<String,Object> params }` |
| `...config.BrowserRequestProperties` | 修改 | 加 `recordingMaxDurationMs` / `recordingMaxCalls` / `responseBodyMaxBytes` / `responseBodyTruncateAt` / `sensitiveKeywords` / `replayStepIntervalMs` |
| `...config.BrowserRequestDescriptor` | 修改 | name 改「站点录制编排」；description 跟着调整 |
| `...config.BrowserRequestAutoConfig` | 修改 | bean 注册：去掉旧 service，加新 service |
| `...config.BrowserSessionManager` | 不变 | |
| `...config.StealthConfig` / `BossRiskBypass` | 不变 | |
| `...domain.BrowserSession` | 不变 | |
| `...domain.SavedRequest` / `BrowserVar` / `Pipeline` / `PipelineRun` | 删除 | |
| `...domain.Recording` | 新建 | record |
| `...domain.HttpCall` | 新建 | record |
| `...domain.Task` | 新建 | record |
| `...domain.TaskRun` | 新建 | record |
| `...domain.enums.RecordingStatus` | 新建 | enum |
| `...domain.enums.TaskRunStatus` | 新建 | enum |
| `...domain.enums.ResourceType` | 新建 | enum |
| `...repository.BrowserSessionRepository` | 不变 | |
| `...repository.SavedRequestRepository` / `BrowserVarRepository` / `PipelineRepository` / `PipelineRunRepository` | 删除 | |
| `...repository.RecordingRepository` | 新建 | |
| `...repository.HttpCallRepository` | 新建 | |
| `...repository.TaskRepository` | 新建 | |
| `...repository.TaskRunRepository` | 新建 | |
| `...service.BrowserRequestService` | 修改 | 收缩为 session 入口 + 委派；删原 execute/save/var/pipeline 相关方法 |
| `...service.CurlParser` / `TemplateRenderer` / `PipelineExecutor` / `JsCapture` | 删除 | |
| `...service.SessionAutoSaver` | 不变 | |
| `...service.SimpleJsonPath` | 保留 | ReplayExecutor 抽取用 |
| `...service.HttpRecorder` | 新建 | 单 ctx 全 HTTP 录制器 |
| `...service.RecordingService` | 新建 | |
| `...service.TaskService` | 新建 | |
| `...service.ReplayExecutor` | 新建 | |
| `...resources.db.browser-request-schema.sql` | 修改 | 删 4 表、加 4 表、保留 session |

### 前端 — TypeScript（`frontend/src/features/browser-request/`）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `index.tsx` | 新建 | FeatureManifest（路由 `/tools/browser-request`） |
| `api.ts` | 新建 | session / recording / task / replay 4 组 fetch 封装 + SSE 工厂 |
| `types.ts` | 新建 | 对齐后端 record（SessionView / RecordingView / HttpCallView / TaskView / TaskRunView / StepSpec / ParameterizationSpec / ExtractSpec / ParamSpec） |
| `pages/BrowserRequestPage.tsx` | 新建 | 顶部 SessionList + 主 Tab(录制/编排/回放) |
| `pages/RecordingDetailPage.tsx` | 新建 | （可选合并到 BrowserRequestPage 录制 tab） |
| `pages/TaskCanvasPage.tsx` | 新建 | 路由 `/tools/browser-request/tasks/:taskId` |
| `pages/ReplayPage.tsx` | 新建 | 路由 `/tools/browser-request/replay/:taskId` |
| `components/SessionList.tsx` | 新建 | 简化版（沿用旧 session 字段） |
| `components/RecordingPanel.tsx` | 新建 | START/STOP + 实时时间线 |
| `components/CallTimeline.tsx` | 新建 | SSE → HttpCallCard 列表 |
| `components/HttpCallCard.tsx` | 新建 | 单 call 卡片（折叠详情） |
| `components/TaskStepEditor.tsx` | 新建 | 单 step：参数化点 + 抽取定义 |
| `components/ParameterizeBubble.tsx` | 新建 | 选区气泡命名变量 |
| `components/ExtractTree.tsx` | 新建 | 响应 JSON 树点字段抽取 |
| `components/ReplayFormDialog.tsx` | 新建 | 填变量 modal |
| `components/ReplayProgressPanel.tsx` | 新建 | SSE 进度面板 |
| `hooks/useRecordingStream.ts` | 新建 | EventSource hook |
| `hooks/useReplayStream.ts` | 新建 | EventSource hook |
| `hooks/useTextSelection.ts` | 新建 | 选区监听 |
| `utils/jsonpath.ts` | 新建 | 子集 JSONPath 自动生成 + 校验 |
| `utils/tokenMatch.ts` | 新建 | parameterization token 校验（恰好出现一次） |

### 关键方法签名与职责

```text
// HttpRecorder（核心）
HttpRecorder#attach(sessionId, recordingId, captureScript): void
  — 在 ctx 上挂 onRequest+onResponse；进入 ACTIVE
HttpRecorder#detach(sessionId): RecordingTermination
  — 摘监听 + 返回终止原因 + 落最终 callCount
HttpRecorder#onResponse(BrowserContext, Response): void
  — Playwright 同步回调；过滤 + 同步取 body（≤256KB）+ 丢异步队列
HttpRecorder#flushAsync(HttpCall): void
  — writer 线程：INSERT call + emit SSE 'call'
HttpRecorder#isSensitive(url, body, sensitiveKeywords): boolean
HttpRecorder#truncateBody(bytes, maxBytes, truncateAt): (text, truncated, dropped)

// RecordingService
RecordingService#start(sessionId, StartRecordingRequest): RecordingView
  — 校 session 已 open；STOP 旧 active；INSERT recording；attach HttpRecorder；起 scheduledTask 超时检测
RecordingService#stop(recordingId, reason): RecordingView
  — detach；UPDATE status/ended_at；emit 'recording-stopped'
RecordingService#listBySession(sessionId): List<RecordingView>
RecordingService#detail(recordingId, withCalls, offset, limit): RecordingDetailView
RecordingService#delete(recordingId): void
  — DELETE calls + DELETE recording；UPDATE task SET recording_id=null
RecordingService#onSessionClosed(sessionId): void
  — 由 BrowserSessionManager.closeSession 钩子触发；自动 STOP active recording

// TaskService
TaskService#create(CreateTaskRequest): TaskView
  — 校 parameterizations：token 必须在对应 field 中恰好出现一次
TaskService#update(taskId, UpdateTaskRequest): TaskView
TaskService#delete(taskId): void  — 级联删 task_run
TaskService#detail(taskId): TaskDetailView
  — steps 中 fromCall 引用已删时取 task 落库的副本 adhoc 部分
TaskService#listBySession(sessionId): List<TaskSummaryView>
TaskService#listRuns(taskId, limit): List<TaskRunSummaryView>
TaskService#runDetail(runId): TaskRunDetailView

TaskService#validateParameterization(token, field, sourceText): void
  — token 不存在 → throw PARAMETERIZATION_TOKEN_NOT_FOUND
  — token 出现多次 → throw PARAMETERIZATION_TOKEN_AMBIGUOUS

// ReplayExecutor
ReplayExecutor#replay(taskId, ReplayRequest): TaskRunView
  — INSERT task_run；异步执行；立即返回 runView
ReplayExecutor#executeAsync(task, taskRun, params): void
  — 顺序执行 steps；每 step 替换参数 → BrowserSessionManager.execute → 抽取 → emit SSE 'step'
ReplayExecutor#renderStep(StepSpec, paramsAndOutputs): ExecuteRequest
  — 把 parameterizations 应用到 url/headers/body：纯字符串替换
ReplayExecutor#extractOutputs(ExecutedResponse, ExtractSpec[]): Map<String,String>
  — 用 SimpleJsonPath 取值；失败 → step 失败 errorMessage="extract X 取到 null"

// Controller（关键端点）
BrowserRequestController#recordingEvents(recordingId): SseEmitter
  — 创建 emitter；registry.register("recording:"+id, emitter)；setTimeout/onError 自动取消
BrowserRequestController#replayEvents(runId): SseEmitter
  — 同上，key = "task-run:"+id

// Repository（关键 SQL；用 JdbcTemplate / Sql2o 或现有 BaseRepository）
RecordingRepository#findActiveBySession(sessionId): Optional<Recording>
RecordingRepository#abandonAllOnStartup(): int  — UPDATE status='ABANDONED' WHERE status='RECORDING'
HttpCallRepository#nextSeq(recordingId): int
HttpCallRepository#findByRecording(recordingId, offset, limit): List<HttpCall>
TaskRepository#findBySessionOrderByUpdatedDesc(sessionId): List<Task>
```

```text
// 前端（关键 hook + 工具）
api.ts#recordings.start(sessionId, body): Promise<RecordingView>
api.ts#recordings.stop(id): Promise<RecordingView>
api.ts#recordings.detail(id, opts): Promise<RecordingDetailView>
api.ts#tasks.create(body): Promise<TaskView>
api.ts#tasks.replay(id, body): Promise<TaskRunView>
api.ts#openRecordingStream(id, handlers): () => void  — 返回 close
api.ts#openReplayStream(runId, handlers): () => void

hooks/useRecordingStream.ts
  — useEffect 挂 EventSource；event 'call' → setCalls(prev => [...prev, view])
  — event 'recording-stopped' → setStatus 终态 + close
hooks/useReplayStream.ts
  — 类似；event 'step' → 累加 stepResults；event 'run-done/failed' → close
hooks/useTextSelection.ts
  — 监听容器内 selectionchange；返回 { selectedText, range, anchorRect }

utils/tokenMatch.ts#countOccurrences(text, token): number
utils/tokenMatch.ts#validateParameterization(token, field, sourceText): { ok, error? }
utils/jsonpath.ts#fromTreePath(path: (string|number)[]): string
  — ['data','items',0,'id'] → "$.data.items[0].id"
utils/jsonpath.ts#getByPath(root, path): unknown  — 前端预览用
```

---

## 4. 数据结构

### 4.1 DB schema（完整 SQL 见 api 文档附录 A）

- 保留：`browser_request_session`
- 新建：`browser_request_recording` / `browser_request_http_call` / `browser_request_task` / `browser_request_task_run`
- 删除（DROP IF EXISTS）：`browser_request_saved` / `browser_request_var` / `browser_request_pipeline` / `browser_request_pipeline_run`
- 关键索引：
  - `idx_recording_active` UNIQUE WHERE status='RECORDING' — 单 session 单 active 守门
  - `idx_call_recording` (recording_id, seq ASC) — 时间线顺序取
  - `idx_task_session` (session_id, updated_at DESC)
  - `idx_run_task` (task_id, started_at DESC)

### 4.2 关键 Java record

```java
// Recording
record Recording(String id, String sessionId, String name,
                 RecordingStatus status, boolean captureScript,
                 long startedAt, Long endedAt, int callCount) {}

// HttpCall（response_headers/body 字段可能很大；查列表时用专门的 SummaryView 不带 body）
record HttpCall(String id, String recordingId, int seq,
                String method, String url, ResourceType resourceType,
                Map<String,String> requestHeaders, String requestBody,
                Integer status,
                Map<String,String> responseHeaders, String responseBody,
                boolean responseTruncated, boolean sensitive,
                long startedAt, Integer elapsedMs, String initiator) {}

// Task
record Task(String id, String sessionId, String recordingId,
            String name,
            List<StepSpec> steps, List<ParamSpec> params,
            TaskOptions options,
            long createdAt, long updatedAt) {}

record StepSpec(String name, String fromCallId, AdhocRequest adhoc,
                List<ParameterizationSpec> parameterizations,
                List<ExtractSpec> extracts, Boolean continueOnError) {}

record ParameterizationSpec(String field, String token, String varName) {}
//   field: url / path / query.{key} / header.{key} / body
record ExtractSpec(String name, String jsonPath) {}
record ParamSpec(String name, String kind, String defaultValue) {}
record TaskOptions(Integer stepIntervalMs, Boolean continueOnError) {}
record AdhocRequest(String method, String url, Map<String,String> headers, String body) {}

// TaskRun
record TaskRun(String id, String taskId, TaskRunStatus status,
               long startedAt, Long finishedAt,
               Map<String,Object> inputs,
               List<StepResult> stepResults,
               String errorMessage) {}

record StepResult(int stepIndex, String stepName, Integer status,
                  Integer elapsedMs, String finalUrl, String responseSample,
                  Map<String,String> extracted, String error) {}

// Enums
enum RecordingStatus { RECORDING, STOPPED, ABANDONED, AUTO_STOPPED }
enum TaskRunStatus { RUNNING, DONE, FAILED, CANCELLED }
enum ResourceType { XHR, FETCH, DOCUMENT, SCRIPT }
```

### 4.3 关键 TS 类型

```ts
type RecordingStatus = 'RECORDING' | 'STOPPED' | 'ABANDONED' | 'AUTO_STOPPED'
type ResourceType = 'XHR' | 'FETCH' | 'DOCUMENT' | 'SCRIPT'
type TaskRunStatus = 'RUNNING' | 'DONE' | 'FAILED' | 'CANCELLED'

interface RecordingView { id, sessionId, name, status, startedAt, endedAt?, callCount, captureScript }
interface HttpCallView {
  id, recordingId, seq, method, url, resourceType,
  status?, elapsedMs?, startedAt,
  requestHeaders?, requestBody?,
  responseHeaders?, responseBody?,
  responseTruncated, sensitive, initiator?
}
interface ParameterizationSpec { field: string; token: string; varName: string }
interface ExtractSpec { name: string; jsonPath: string }
interface ParamSpec { name: string; kind: 'string'|'number'|'boolean'; defaultValue?: string }
interface StepSpec {
  name: string;
  fromCallId?: string;
  adhoc?: { method, url, headers, body }
  parameterizations?: ParameterizationSpec[];
  extracts?: ExtractSpec[];
  continueOnError?: boolean;
}
interface TaskView { id, sessionId, recordingId?, name, steps, params, options, createdAt, updatedAt }
interface StepResultView { stepIndex, stepName, status?, elapsedMs?, extracted, error? }
interface TaskRunView { id, taskId, status, startedAt, finishedAt?, inputs, stepResults, errorMessage? }
```

---

## 5. 重要约束与边界

- **并发**：单 session 同时只能 1 个 active recording（DB 部分 UNIQUE 索引 + 应用层 START 前 STOP 旧的）
- **并发**：单 session 同时只能 1 个 RUNNING task_run（Playwright worker 单线程决定，无需 DB 锁）
- **事务**：录制中每条 HttpCall INSERT 是独立事务；recording.callCount 同事务里 +1
- **事务**：task_run 进度更新是「整 step 完成才 UPDATE 一次 stepResultsJson」，避免每个事件都写 DB
- **HttpRecorder writer 线程**：单 ExecutorService，串行落库 + 串行推 SSE（防止 SQLite 并发写冲突 + 防 SSE 乱序）
- **SSE key 规约**：录制 = `"recording:" + recordingId`，回放 = `"task-run:" + runId`
- **资源回收**：每个 SseEmitter 挂 `onCompletion/onTimeout/onError` 自动从 registry 移除
- **app 关闭路径**：`@PreDestroy` → 停所有 HttpRecorder → 关 Playwright → 进程关
- **后端启动路径**：`@PostConstruct` 或 `ApplicationReadyEvent` → `RecordingRepository.abandonAllOnStartup()`

---

## 6. 下游依赖调用

- Playwright（已有依赖，不新增）：
  - `BrowserContext.onRequest / onResponse`：录制监听
  - `BrowserContext.request()`（APIRequestContext）：回放执行（沿用 `BrowserSessionManager.execute`）
- toolbox 内部：
  - `SseEmitterRegistry`（已有）：SSE emitter 1:1 key→emitter 注册

---

## 7. 异常处理要点

| 场景 | 处理 |
|------|------|
| session 未 open 调用 start/replay | 抛 `IllegalStateException("会话未打开")`；Controller 转 409 `SESSION_NOT_OPEN` |
| 已有 active recording 时调 start | 自动 STOP 旧的（不抛错），再创建新 recording |
| 重复 STOP | 幂等：直接返回当前 RecordingView |
| parameterization token 校验失败 | 抛 `BadRequestException` → 400 `PARAMETERIZATION_TOKEN_NOT_FOUND` / `PARAMETERIZATION_TOKEN_AMBIGUOUS` |
| replay 时缺变量 | `MissingParamException`，错误信息精确到 "缺 docId"；step 失败而非整 task 失败（让用户在 stepResults 里看到上下文） |
| extract jsonPath 取到 null | step 失败 `error="extract X 取到 null"`；该 step 后续 step 引用 X 时同样失败 |
| Playwright fetch 异常 | `step.status=null, error="fetch failed: " + ex.message`；按 continueOnError 决定是否继续 |
| HttpRecorder writer 异常 | 单条 INSERT 失败 → log.warn，不抛出，保持后续 call 不丢；SSE 推送失败 → log.debug，不影响落库 |
| SQLite 锁等待 | writer 是单线程，理论无锁等待；如果用 WAL 模式但仍发生 BUSY，重试 3 次后丢弃这一条（log.warn） |
| SSE 客户端断开 | onError/onTimeout 自动 unregister；下次同 recording 重连时拉历史 calls（前端用 GET /recordings/{id}?withCalls 补齐再切流） |

---

## 8. 实施顺序（建议）

1. **DB schema 重做 + 旧表删除**（一次性切换；首次启动时 DROP 旧表 + CREATE 新表）
2. **后端 domain + repository + DTO**（先把数据层骨架立起来）
3. **HttpRecorder + RecordingService**（核心录制能力，先跑通"录制+落库+SSE"）
4. **TaskService**（参数化校验是关键，要先单元测试覆盖）
5. **ReplayExecutor**（依赖 1-4 全部就绪）
6. **Controller 改造**（删旧端点 + 加新端点 + SSE）
7. **前端 types.ts + api.ts**（先把契约对齐）
8. **前端 SessionList + RecordingPanel**（最小可用：能开/关录制 + 看到时间线）
9. **前端 TaskCanvas**（编排，复杂但独立）
10. **前端 ReplayPanel**（最后；依赖 1-9 全部跑通）
11. **typecheck + dev server + 手工冒烟**
