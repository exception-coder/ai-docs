# 视频名称归类（接口契约）

> 配套：[视频名称归类-current.md](视频名称归类-current.md)

## 接口清单

| Method | Path | 说明 |
|--------|------|------|
| POST | `/api/treesize/videos/name-grouping/start` | 启动归类任务 |
| POST | `/api/treesize/videos/name-grouping/stop` | 取消 |
| GET | `/api/treesize/videos/name-grouping/status` | 任务状态 |
| GET | `/api/treesize/videos/name-grouping/events` | SSE 进度 |
| GET | `/api/treesize/videos/series/{signature}` | （**新增**）查询同系列的所有视频 |

前 4 个接口结构与「视频语言识别」同构；type=NAME_GROUPING。

## GET `/api/treesize/videos/series/{signature}`

返回与给定 signature 相同的所有视频，按 `series_episode` 升序（NULL 在末尾），再按 name 升序。

**响应 200**：

```json
{
  "signature": "进击的巨人",
  "count": 25,
  "items": [
    { "path": "...", "name": "...", "size": ..., "episode": 1, "thumbnailGridPath": "...", ... },
    { "path": "...", "name": "...", "size": ..., "episode": 2, ... }
  ]
}
```

**响应 404**：signature 不存在任何匹配视频。

## 调用示例

```typescript
export function startNameGrouping() {
  return http<{ jobId: string; total: number }>('/treesize/videos/name-grouping/start', { method: 'POST' })
}
export function stopNameGrouping() {
  return http<void>('/treesize/videos/name-grouping/stop', { method: 'POST' })
}
export function getNameGroupingStatus() {
  return http<ProcessingJob | null>('/treesize/videos/name-grouping/status')
}
export function nameGroupingEventsPath() {
  return '/treesize/videos/name-grouping/events'
}
export function getVideoSeries(signature: string) {
  return http<SeriesView>(`/treesize/videos/series/${encodeURIComponent(signature)}`)
}

export interface SeriesView {
  signature: string
  count: number
  items: Array<{
    path: string
    name: string
    size: number
    episode: number | null
    thumbnailGridPath: string | null
    // ...其它 VideoLibraryItem 字段
  }>
}
```
