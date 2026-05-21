# 视频人物年龄识别（接口契约）

> 配套：[视频人物年龄识别-current.md](视频人物年龄识别-current.md)

## 接口清单

| Method | Path | 说明 |
|--------|------|------|
| POST | `/api/treesize/videos/person-age/start` | 启动识别任务 |
| POST | `/api/treesize/videos/person-age/stop` | 取消 |
| GET | `/api/treesize/videos/person-age/status` | 任务状态 |
| GET | `/api/treesize/videos/person-age/events` | SSE 进度 |

接口结构与「视频语言识别」完全同构，差别只在 path 与 type=PERSON_AGE_DETECT；详见 [视频语言识别-api-current.md](../视频语言识别/视频语言识别-api-current.md) 套用。

## 出参枚举

`person_main_age_group`：

| 值 | 含义 |
|----|------|
| `no_person` | 未检测到人物 |
| `infant` | 0-3 岁 |
| `child` | 4-12 岁 |
| `teen` | 13-19 岁 |
| `young_adult` | 20-35 岁 |
| `middle_age` | 36-55 岁 |
| `senior` | 56+ 岁 |

`person_main_gender`：`M` / `F` / `unknown`

## 调用示例

```typescript
export interface PersonAgeJob {
  id: string
  type: 'PERSON_AGE_DETECT'
  status: 'RUNNING' | 'DONE' | 'FAILED' | 'CANCELLED'
  total: number; processed: number; succeeded: number; failed: number
  currentPath: string | null; errorMsg: string | null
  startedAt: number; finishedAt: number | null
}

export function startPersonAgeDetect() {
  return http<{ jobId: string; total: number }>('/treesize/videos/person-age/start', { method: 'POST' })
}
export function stopPersonAgeDetect() {
  return http<void>('/treesize/videos/person-age/stop', { method: 'POST' })
}
export function getPersonAgeStatus() {
  return http<PersonAgeJob | null>('/treesize/videos/person-age/status')
}
export function personAgeEventsPath() {
  return '/treesize/videos/person-age/events'
}
```
