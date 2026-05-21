# 视频表与同步功能（接口契约）

> 最后更新：2026-05-21
> 配套设计文档：[视频表与同步功能-current.md](视频表与同步功能-current.md)

## 接口清单

| Method | Path | 说明 | 实现位置 |
|--------|------|------|---------|
| POST | `/api/treesize/videos/sync` | 从 `treesize_node` 同步视频到 `treesize_video` | `TreeSizeController#syncVideos` |

---

## POST `/api/treesize/videos/sync`

### 描述

把 `treesize_node` 表中所有已扫描的视频文件汇总到独立的 `treesize_video` 表，作为视频库后续语言识别、媒体属性填充、列表数据源的基础设施。

执行语义：**只增不改**。已存在于 `treesize_video` 的 `path` 跳过（保护已填字段不被覆盖），新 path 插入。

### 请求

- **方法**：`POST`
- **路径**：`/api/treesize/videos/sync`
- **入参**：无（不带 query / body）
- **认证**：与其他 treesize 接口一致（继承全局）

### 响应

**200 OK**

```json
{
  "scannedFromNode": 1842,
  "insertedNew": 1842,
  "skippedExisting": 0,
  "skippedTooSmall": 37,
  "elapsedMs": 412
}
```

#### 响应字段

| 字段 | 类型 | 含义 |
|------|------|------|
| `scannedFromNode` | `integer` | 通过 SQL 过滤（ext + size>=100KB）后扫描到的视频节点数；等于 `insertedNew + skippedExisting` |
| `insertedNew` | `integer` | 本次新插入到 `treesize_video` 的行数 |
| `skippedExisting` | `integer` | `path` 已存在被跳过的行数 |
| `skippedTooSmall` | `integer` | 大小 < 100KB 被过滤掉的视频文件数（信息性字段，让用户感知噪音规模） |
| `elapsedMs` | `integer` | 同步耗时，毫秒 |

#### 字段约束

- 所有计数字段 ≥ 0
- `scannedFromNode = insertedNew + skippedExisting`（恒等）
- `elapsedMs` 仅供展示，不参与判断

### 错误码

| 状态码 | 场景 | 响应体 |
|--------|------|--------|
| 500 | SQLite 锁定 / 表不存在 / IO 异常 | `{ "message": "…" }` 用项目通用错误体 |

### 调用示例

```bash
curl -X POST http://localhost:8080/api/treesize/videos/sync
```

```typescript
// frontend/src/features/video-library/api.ts
export function syncVideoLibrary() {
  return http<VideoSyncResult>('/treesize/videos/sync', { method: 'POST' })
}

export interface VideoSyncResult {
  scannedFromNode: number
  insertedNew: number
  skippedExisting: number
  skippedTooSmall: number
  elapsedMs: number
}
```

### 幂等性

- 幂等。重复点击不会产生副作用（`INSERT OR IGNORE`）
- 第二次调用通常 `insertedNew=0` / `skippedExisting=全量`，除非期间又有新扫盘把新视频写进了 `treesize_node`

### 性能预期

| 视频量级 | 预期耗时 |
|---------|---------|
| 1,000 | < 100ms |
| 10,000 | < 1s |
| 100,000 | < 10s |

> 同步阻塞执行，前端按钮在请求期间禁用 + Loading 状态。超过 10 秒级时考虑下期改 SSE。
