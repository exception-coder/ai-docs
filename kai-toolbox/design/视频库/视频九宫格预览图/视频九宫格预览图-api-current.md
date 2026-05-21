# 视频九宫格预览图（接口契约）

> 最后更新：2026-05-21
> 配套：[视频九宫格预览图-current.md](视频九宫格预览图-current.md)

## 接口清单

| Method | Path | 说明 | 实现位置 |
|--------|------|------|---------|
| POST | `/api/treesize/videos/thumbnail-grid/start` | 启动九宫格生成任务 | `TreeSizeController#startThumbnailGrid` |
| POST | `/api/treesize/videos/thumbnail-grid/stop` | 取消当前任务 | `TreeSizeController#stopThumbnailGrid` |
| GET | `/api/treesize/videos/thumbnail-grid/status` | 查询当前 / 最近一次任务 | `TreeSizeController#getThumbnailGridStatus` |
| GET | `/api/treesize/videos/thumbnail-grid/events` | SSE 进度事件流 | `TreeSizeController#thumbnailGridEvents` |
| GET | `/api/treesize/videos/thumbnail-grid` | 取某视频的九宫格 JPEG | `TreeSizeController#getThumbnailGrid` |

---

## POST `/start`

**响应 200**：`{ "jobId": "uuid-...", "total": 1842 }`

**响应 409**（已有 RUNNING）：`{ "message": "...", "jobId": "uuid-of-running" }`

**响应 500**（致命）：`{ "message": "..." }`

---

## POST `/stop`

**响应 204**。幂等。

---

## GET `/status`

返回最近一次 `type='THUMBNAIL_GRID'` 的任务行，与语言识别 status 同结构（仅 `type` 不同）。

```json
{
  "id": "uuid-...",
  "type": "THUMBNAIL_GRID",
  "status": "RUNNING",
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

---

## GET `/events` (SSE)

事件结构与语言识别 SSE 完全一致：

- `event: progress` — 每完成 1 个视频推一次
- `event: done` — 任务终止（DONE / CANCELLED / FAILED）

---

## GET `/thumbnail-grid?path=<urlencoded>`

**入参**：query `path`，绝对路径（URL encode）

**响应 200**：`Content-Type: image/jpeg`，body 为 JPEG 二进制流。`Cache-Control: max-age=86400` 可选（图片内容固定）

**响应 404**：
- 数据库无对应 video 行
- video 行的 `thumbnail_grid_path` 为 NULL（还没生成）
- 缓存文件被删

**响应 500**：IO 错误（理论上不应该发生，磁盘问题除外）

---

## 调用示例

```typescript
// frontend/src/features/video-library/api.ts
export interface ThumbnailGridJob {
  id: string
  type: 'THUMBNAIL_GRID'
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

export function startThumbnailGrid() {
  return http<{ jobId: string; total: number }>('/treesize/videos/thumbnail-grid/start', { method: 'POST' })
}
export function stopThumbnailGrid() {
  return http<void>('/treesize/videos/thumbnail-grid/stop', { method: 'POST' })
}
export function getThumbnailGridStatus() {
  return http<ThumbnailGridJob | null>('/treesize/videos/thumbnail-grid/status')
}
export function thumbnailGridEventsPath() {
  return '/treesize/videos/thumbnail-grid/events'
}

/** Absolute URL for <img src>. */
export function thumbnailGridUrl(path: string): string {
  return `/api/treesize/videos/thumbnail-grid?path=${encodeURIComponent(path)}`
}
```
