# 智能加速下载器 · 接口文档

> 最后更新：2026-05-25
> 配套设计文档：[智能加速下载器-current.md](智能加速下载器-current.md)

---

## 响应格式约定

> 与 kai-toolbox 其他工具一致，**成功响应直接返回 DTO body，无 `code/data` 包装**。错误响应由 `com.exceptioncoder.toolbox.common.exception.GlobalExceptionHandler` 统一包装为：
>
> ```json
> { "timestamp": "2026-05-25T10:30:00Z", "status": 400, "error": "Bad Request", "message": "url: must not be blank" }
> ```

---

## 接口清单

| # | 方法 | 路径 | 用途 |
|---|------|------|------|
| 1 | POST | /api/downloader/tasks | 创建新下载任务 |
| 2 | GET | /api/downloader/tasks | 列出所有任务（活动 + 历史） |
| 3 | GET | /api/downloader/tasks/{id} | 任务详情（含分片明细 + 链路决策） |
| 4 | POST | /api/downloader/tasks/{id}/pause | 暂停任务 |
| 5 | POST | /api/downloader/tasks/{id}/resume | 恢复任务 |
| 6 | DELETE | /api/downloader/tasks/{id} | 删除任务（可选保留临时文件） |
| 7 | GET | /api/downloader/tasks/{id}/events | SSE 进度推送 |
| 8 | GET | /api/downloader/proxy/detect | 手动触发一次系统代理探测 |

---

## 1. 创建下载任务

- **方法**：`POST`
- **路径**：`/api/downloader/tasks`
- **认证**：不需要
- **幂等**：否

### 1.1 Body

```json
{
  "url": "https://dl.feishu.cn/.../Lark-win32_x64-7.67.10-signed.exe",
  "savePath": null,
  "filename": null
}
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `url` | string | 是 | - | HTTP 或 HTTPS 直链；3xx 重定向最多跟随 5 次 |
| `savePath` | string | 否 | `<user.home>/Downloads/kai-toolbox` | 保存目录，不存在自动创建；不允许写到系统目录 |
| `filename` | string | 否 | 从 URL 末段或 `Content-Disposition` 推断 | 自动 sanitize Windows 非法字符 |

### 1.2 成功响应（201）

```json
{
  "taskId": 42,
  "url": "https://dl.feishu.cn/.../Lark-win32_x64-7.67.10-signed.exe",
  "savePath": "C:/Users/zhang/Downloads/kai-toolbox",
  "filename": "Lark-win32_x64-7.67.10-signed.exe",
  "totalSize": 368050176,
  "downloadedSize": 0,
  "state": "QUEUED",
  "routeType": null,
  "routeProxy": null,
  "currentRateBps": 0,
  "etaSeconds": null,
  "createdAt": "2026-05-25T10:30:00Z",
  "updatedAt": "2026-05-25T10:30:00Z"
}
```

### 1.3 错误响应

| HTTP | message 范例 | 触发场景 |
|------|-------------|---------|
| 400 | `url: must not be blank` / `url 必须以 http(s):// 开头` | 参数校验失败 |
| 400 | `savePath 不允许写入系统目录` | savePath 非法 |
| 422 | `URL 需要登录认证，暂不支持` | HEAD 返回 401/403/重定向到登录页 |
| 502 | `两条链路均无法访问目标 URL` | HEAD 探测在两条链路都超时 |

---

## 2. 列出任务

- **方法**：`GET`
- **路径**：`/api/downloader/tasks`

### 2.1 Query

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `state` | string | 否 | 按状态过滤；多值逗号分隔，如 `state=DOWNLOADING,QUEUED` |
| `limit` | integer | 否 | 默认 50，最大 200 |

### 2.2 成功响应（200）

```json
[
  {
    "taskId": 42,
    "url": "https://...",
    "savePath": "C:/Users/zhang/Downloads/kai-toolbox",
    "filename": "Lark-win32_x64-7.67.10-signed.exe",
    "totalSize": 368050176,
    "downloadedSize": 184025088,
    "state": "DOWNLOADING",
    "routeType": "PROXY",
    "routeProxy": "http://127.0.0.1:7890",
    "currentRateBps": 12582912,
    "etaSeconds": 14,
    "createdAt": "2026-05-25T10:30:00Z",
    "updatedAt": "2026-05-25T10:30:42Z"
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `totalSize` | long | 总字节；未拿到 `Content-Length` 时为 `-1` |
| `state` | string | `QUEUED` / `PROBING` / `DOWNLOADING` / `PAUSED` / `COMPLETED` / `FAILED` |
| `routeType` | string\|null | `DIRECT` / `PROXY`；未探测前为 `null` |
| `currentRateBps` | long | 最近 500ms 平均速率（字节/秒） |
| `etaSeconds` | long\|null | 预估剩余秒数；totalSize 未知时为 `null` |

---

## 3. 任务详情

- **方法**：`GET`
- **路径**：`/api/downloader/tasks/{id}`

### 3.1 成功响应（200）

```json
{
  "taskId": 42,
  "url": "https://...",
  "savePath": "C:/Users/zhang/Downloads/kai-toolbox",
  "filename": "Lark-win32_x64-7.67.10-signed.exe",
  "totalSize": 368050176,
  "acceptRanges": true,
  "downloadedSize": 184025088,
  "state": "DOWNLOADING",
  "routeDecision": {
    "routeType": "PROXY",
    "routeProxy": "http://127.0.0.1:7890",
    "directTtfbMs": 1200,
    "directThroughputBps": 56000,
    "proxyTtfbMs": 280,
    "proxyThroughputBps": 320000,
    "decidedAt": "2026-05-25T10:30:02Z"
  },
  "segments": [
    { "seqNo": 0, "offset": 0, "length": 33554432, "bytesDownloaded": 33554432, "state": "DONE",        "attempts": 1 },
    { "seqNo": 1, "offset": 33554432, "length": 33554432, "bytesDownloaded": 12345678, "state": "DOWNLOADING", "attempts": 1 }
  ],
  "lastError": null,
  "createdAt": "2026-05-25T10:30:00Z",
  "updatedAt": "2026-05-25T10:30:42Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `acceptRanges` | boolean | 服务端是否支持 `Accept-Ranges: bytes` |
| `routeDecision` | object\|null | race 决策详情；尚未探测时为 `null` |
| `segments[].state` | string | `PENDING` / `DOWNLOADING` / `DONE` / `FAILED` |
| `lastError` | string\|null | 任务级最后错误（state=FAILED 时填写） |

### 3.2 错误响应

| HTTP | 触发场景 |
|------|---------|
| 404 | 任务 ID 不存在 |

---

## 4. 暂停任务

- **方法**：`POST`
- **路径**：`/api/downloader/tasks/{id}/pause`
- **幂等**：是

### 4.1 成功响应（200）

返回与「2. 列出任务」单条相同的 `TaskView`，`state` 字段为 `PAUSED`。

### 4.2 错误响应

| HTTP | 触发场景 |
|------|---------|
| 404 | 任务 ID 不存在 |

---

## 5. 恢复任务

- **方法**：`POST`
- **路径**：`/api/downloader/tasks/{id}/resume`
- **行为**：沿用上次链路决策，**不重新探测**；从已下载偏移继续

### 5.1 成功响应（200）

返回 `TaskView`，`state` 通常为 `DOWNLOADING`。

### 5.2 错误响应

| HTTP | message 范例 | 触发场景 |
|------|-------------|---------|
| 404 | `task not found: 42` | 任务不存在 |
| 409 | `任务当前状态不可恢复：COMPLETED` | COMPLETED / 已 DOWNLOADING |

---

## 6. 删除任务

- **方法**：`DELETE`
- **路径**：`/api/downloader/tasks/{id}`
- **幂等**：是

### 6.1 Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `keepFile` | boolean | 否 | `false` | `true` 时即使未完成也保留临时文件，仅删数据库记录 |

### 6.2 成功响应（204）

无 body。

---

## 7. 任务事件流（SSE）

- **方法**：`GET`
- **路径**：`/api/downloader/tasks/{id}/events`
- **协议**：Server-Sent Events
- **心跳**：底层 emitter 超时 60 分钟（toolbox-common `SseEmitterRegistry` 默认）；窗口聚合期间无活动会自动发空 progress 保活

### 7.1 事件类型

#### `progress`

> 触发：每 500ms 聚合一次（仅在 DOWNLOADING 时推送）

```
event: progress
data: {"taskId":42,"downloaded":184025088,"total":368050176,"rateBps":12582912,"etaSeconds":14}
```

#### `state`

> 触发：任务状态切换时立即推送

```
event: state
data: {"taskId":42,"state":"DOWNLOADING","routeType":"PROXY","routeProxy":"http://127.0.0.1:7890","error":null}
```

#### `segment`

> 触发：任意分片状态切换

```
event: segment
data: {"taskId":42,"seqNo":3,"state":"DONE","attempts":1,"bytesDownloaded":33554432}
```

### 7.2 关闭

- 任务终态（`COMPLETED` / `FAILED`）后服务端发送对应 `state` 事件并调用 `SseEmitterRegistry.complete(taskId)`，客户端 `EventSource.close()`

---

## 8. 系统代理探测

- **方法**：`GET`
- **路径**：`/api/downloader/proxy/detect`

### 8.1 成功响应（200）

```json
{
  "candidates": [
    { "source": "ENV",              "type": "HTTP", "host": "127.0.0.1", "port": 7890, "originUrl": "http://127.0.0.1:7890" },
    { "source": "WINDOWS_REGISTRY", "type": "HTTP", "host": "127.0.0.1", "port": 7890, "originUrl": "http://127.0.0.1:7890" }
  ],
  "effective": {
    "source": "ENV",
    "type": "HTTP",
    "host": "127.0.0.1",
    "port": 7890,
    "originUrl": "http://127.0.0.1:7890"
  },
  "detectedAt": "2026-05-25T10:29:55Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `candidates[].source` | string | `JVM_PROPERTY` / `ENV` / `WINDOWS_REGISTRY` / `TOOLBOX_CONFIG` |
| `effective` | object\|null | 最终使用的代理；无代理时为 `null` |
