# 视频嵌入与相似聚类（接口契约）

> 配套：[视频嵌入与相似聚类-current.md](视频嵌入与相似聚类-current.md)

## 接口清单

### 嵌入生成任务（后台）

| Method | Path | 说明 |
|--------|------|------|
| POST | `/api/treesize/videos/visual/embed/start` | 启动 |
| POST | `/api/treesize/videos/visual/embed/stop` | 取消 |
| GET | `/api/treesize/videos/visual/embed/status` | 状态 |
| GET | `/api/treesize/videos/visual/embed/events` | SSE 进度 |

结构与其他任务同构；type=VISUAL_EMBED。

### 聚类任务（one-shot）

| Method | Path | 说明 |
|--------|------|------|
| POST | `/api/treesize/videos/visual/cluster/start` | 启动 |
| GET | `/api/treesize/videos/visual/cluster/status` | 状态 |

无 stop 端点（聚类是单步操作，运行中无法中断）。type=VISUAL_CLUSTER。

### 相似检索（同步 HTTP）

| Method | Path | 说明 |
|--------|------|------|
| GET | `/api/treesize/videos/visual/similar` | 查相似视频 |

**入参**：
- `path` (string, required)：基准视频的绝对路径，URL encode
- `topN` (int, optional, default=10)：返回前 N 个
- `minSimilarity` (float, optional, default=0.0)：相似度下限阈值

**响应 200**：
```json
{
  "queryPath": "D:\\videos\\target.mp4",
  "results": [
    { "path": "D:\\...", "name": "...", "thumbnailGridPath": "...", "similarity": 0.92 },
    { "path": "D:\\...", "name": "...", "thumbnailGridPath": "...", "similarity": 0.88 }
  ]
}
```

**响应 404**：基准视频无 embedding。

### 聚类查询

| Method | Path | 说明 |
|--------|------|------|
| GET | `/api/treesize/videos/visual/clusters` | 列出全部聚类 + 每簇代表 |
| GET | `/api/treesize/videos/visual/cluster/{id}` | 列出指定聚类的所有视频 |

**`/visual/clusters` 响应 200**：

```json
{
  "totalClusters": 12,
  "noiseCount": 47,
  "clusters": [
    {
      "id": 0,
      "label": null,
      "count": 23,
      "representative": { "path": "...", "name": "...", "thumbnailGridPath": "..." }
    },
    ...
  ]
}
```

`representative` 选 size 最大的一个视频作为该簇代表。

## 调用示例

```typescript
export interface SimilarVideo {
  path: string
  name: string
  thumbnailGridPath: string | null
  similarity: number   // 0..1
}

export function startVisualEmbed() {
  return http<{ jobId: string; total: number }>('/treesize/videos/visual/embed/start', { method: 'POST' })
}
export function stopVisualEmbed() {
  return http<void>('/treesize/videos/visual/embed/stop', { method: 'POST' })
}
export function getVisualEmbedStatus() {
  return http<ProcessingJob | null>('/treesize/videos/visual/embed/status')
}

export function startVisualCluster() {
  return http<{ jobId: string }>('/treesize/videos/visual/cluster/start', { method: 'POST' })
}
export function getVisualClusterStatus() {
  return http<ProcessingJob | null>('/treesize/videos/visual/cluster/status')
}

export function getSimilarVideos(path: string, topN = 10, minSimilarity = 0) {
  const qs = new URLSearchParams({
    path,
    topN: String(topN),
    minSimilarity: String(minSimilarity),
  })
  return http<{ queryPath: string; results: SimilarVideo[] }>(
    `/treesize/videos/visual/similar?${qs}`)
}

export function listClusters() {
  return http<{ totalClusters: number; noiseCount: number; clusters: ClusterView[] }>(
    '/treesize/videos/visual/clusters')
}
```
