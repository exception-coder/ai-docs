# 视频语言识别（接口契约）

> 最后更新：2026-05-21
> 配套：[视频语言识别-current.md](视频语言识别-current.md)

## 接口清单

| Method | Path | 说明 | 实现位置 |
|--------|------|------|---------|
| POST | `/api/treesize/videos/language-detect/start` | 启动语言识别任务 | `TreeSizeController#startLanguageDetect` |
| POST | `/api/treesize/videos/language-detect/stop` | 取消当前语言识别任务 | `TreeSizeController#stopLanguageDetect` |
| GET | `/api/treesize/videos/language-detect/status` | 查询当前 / 最近一次任务状态 | `TreeSizeController#getLanguageDetectStatus` |
| GET | `/api/treesize/videos/language-detect/events` | SSE 进度事件流 | `TreeSizeController#languageDetectEvents` |

---

## POST `/start`

**请求**：无 body

**响应 200**：
```json
{ "jobId": "uuid-...", "total": 1842 }
```

**响应 409**（已有 RUNNING 任务）：
```json
{ "message": "language detect task already running", "jobId": "uuid-of-running" }
```

**响应 500**（致命错误：Whisper 二进制不存在 / dry-run 失败）：
```json
{ "message": "..." }
```

---

## POST `/stop`

**请求**：无 body

**响应 204**：成功（无论本来有没有 RUNNING 任务都返回 204；幂等）

---

## GET `/status`

**响应 200**：返回最近一次任务（按 `started_at DESC LIMIT 1` 查 `type='LANGUAGE_DETECT'`）

```json
{
  "id": "uuid-...",
  "type": "LANGUAGE_DETECT",
  "status": "RUNNING",          // RUNNING | DONE | FAILED | CANCELLED
  "total": 1842,
  "processed": 423,
  "succeeded": 418,
  "failed": 5,
  "currentPath": "D:\\videos\\movie.mp4",
  "errorMsg": null,
  "startedAt": 1716265200000,
  "finishedAt": null
}
```

**响应 200（无任何历史任务）**：`null`

---

## GET `/events`（SSE）

**协议**：`text/event-stream`，标准 SSE 格式

**事件类型**：

### `progress`（每完成 1 个视频推一次）

```
event: progress
data: {"jobId":"uuid","processed":424,"succeeded":419,"failed":5,"currentPath":"D:\\videos\\next.mp4"}
```

### `done`（任务终止）

```
event: done
data: {"jobId":"uuid","status":"DONE","processed":1842,"succeeded":1820,"failed":22}
```

**status** 取值：`DONE` | `CANCELLED` | `FAILED`

### `error`（SSE 链路错误）

由 Spring SseEmitter 标准机制处理，前端 EventSource onError 重连即可。

---

## 字段约束

| 字段 | 类型 | 约束 |
|------|------|------|
| `jobId` | string(UUID) | |
| `type` | enum | `LANGUAGE_DETECT` |
| `status` | enum | `RUNNING / DONE / FAILED / CANCELLED` |
| `total` | int ≥ 0 | 启动时快照 |
| `processed` | int ≥ 0 | `≤ total`，但极端情况（任务跑中又有视频被同步进来）可能 `> total`，前端不要做硬等式判断 |
| `succeeded` | int ≥ 0 | |
| `failed` | int ≥ 0 | `processed = succeeded + failed` |
| `currentPath` | string \| null | RUNNING 时实时刷新；终止后可保留最后一个 |
| `errorMsg` | string \| null | 最近一次失败错误，便于 debug |

---

## 调用示例

```typescript
// frontend/src/features/video-library/api.ts
export interface LanguageDetectJob {
  id: string
  type: 'LANGUAGE_DETECT'
  status: 'RUNNING' | 'DONE' | 'FAILED' | 'CANCELLED'
  total: number
  processed: number
  succeeded: number
  failed: number
  currentPath: string | null
  errorMsg: string | null
  startedAt: number
  finishedAt: number | null
}

export function startLanguageDetect() {
  return http<{ jobId: string; total: number }>('/treesize/videos/language-detect/start', { method: 'POST' })
}
export function stopLanguageDetect() {
  return http<void>('/treesize/videos/language-detect/stop', { method: 'POST' })
}
export function getLanguageDetectStatus() {
  return http<LanguageDetectJob | null>('/treesize/videos/language-detect/status')
}
export function languageDetectEventsPath() {
  return '/treesize/videos/language-detect/events'   // 配 subscribeSse 复用现有 SSE 客户端
}
```
