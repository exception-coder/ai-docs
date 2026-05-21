# 视频时长区间分类（接口契约）

> 配套：[视频时长区间分类-current.md](视频时长区间分类-current.md)

## 接口清单

| Method | Path | 说明 |
|--------|------|------|
| POST | `/api/treesize/videos/duration-probe/start` | 启动 |
| POST | `/api/treesize/videos/duration-probe/stop` | 取消 |
| GET | `/api/treesize/videos/duration-probe/status` | 任务状态 |
| GET | `/api/treesize/videos/duration-probe/events` | SSE 进度 |

结构与「视频语言识别」同构；type=DURATION_PROBE。详见 [视频语言识别-api-current.md](../视频语言识别/视频语言识别-api-current.md)。

## 出参枚举

`duration_bucket`：

| 值 | 含义 |
|----|------|
| `micro` | < 30s |
| `short` | 30s ~ 5min |
| `medium` | 5min ~ 30min |
| `long` | 30min ~ 90min |
| `xlong` | > 90min |
| `unknown` | 探测失败 |

## 调用示例

```typescript
export function startDurationProbe() {
  return http<{ jobId: string; total: number }>('/treesize/videos/duration-probe/start', { method: 'POST' })
}
export function stopDurationProbe() {
  return http<void>('/treesize/videos/duration-probe/stop', { method: 'POST' })
}
export function getDurationProbeStatus() {
  return http<ProcessingJob | null>('/treesize/videos/duration-probe/status')
}
export function durationProbeEventsPath() {
  return '/treesize/videos/duration-probe/events'
}
```
