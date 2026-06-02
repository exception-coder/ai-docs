# Docker 服务治理 - 接口文档

> 与 `Docker服务治理-current.md`（技术方案）配套，字段级契约只在本文档维护。
> 最后更新：2026-05-27

---

## 接口清单

| #  | 方法   | 路径 | 用途 |
|----|--------|------|------|
| 1  | GET    | `/api/docker/hosts/{hostId}/apps` | 列出某主机已登记的 Docker 应用 |
| 2  | POST   | `/api/docker/hosts/{hostId}/apps` | 新增应用登记 |
| 3  | PUT    | `/api/docker/hosts/{hostId}/apps/{appId}` | 修改应用登记 |
| 4  | DELETE | `/api/docker/hosts/{hostId}/apps/{appId}` | 删除应用登记（不动远端） |
| 5  | POST   | `/api/docker/hosts/{hostId}/scan` | 扫描指定根目录发现 compose 应用 |
| 6  | GET    | `/api/docker/hosts/{hostId}/containers` | 列出主机所有容器（可按 appId 过滤） |
| 7  | POST   | `/api/docker/hosts/{hostId}/containers/{cid}/{action}` | 单容器动作（start/stop/restart/pause/unpause/kill） |
| 8  | GET    | `/api/docker/hosts/{hostId}/containers/stats` | 全主机一次性资源快照 |
| 9  | POST   | `/api/docker/hosts/{hostId}/apps/{appId}/compose/{action}` | compose 整体动作（up/down/restart/pull） |
| 10 | GET    | `/api/docker/hosts/{hostId}/containers/{cid}/logs` | 一次性 tail 容器日志 |
| 11 | GET    | `/api/docker/hosts/{hostId}/containers/{cid}/logs/stream` | SSE 流式 follow 日志 |
| 12 | DELETE | `/api/docker/streams/{streamId}` | 主动关闭日志流 |
| 13 | GET    | `/api/docker/hosts/{hostId}/apps/{appId}/files` | 列应用目录下白名单配置文件 |
| 14 | GET    | `/api/docker/hosts/{hostId}/apps/{appId}/files/content` | 读单个配置文件内容 |
| 15 | PUT    | `/api/docker/hosts/{hostId}/apps/{appId}/files/content` | 保存单个配置文件（自动备份） |

**通用错误体（所有 4xx/5xx）：**

```json
{ "error": "BAD_REQUEST", "message": "可读错误描述" }
```

**通用枚举：**
- `containerAction` ∈ `start | stop | restart | pause | unpause | kill`
- `composeAction` ∈ `up | down | restart | pull`

---

## 1. 列应用

### 1.1 基本信息
- **方法**：`GET`
- **路径**：`/api/docker/hosts/{hostId}/apps`
- **认证**：不需要

### 1.2 请求

| 字段 | 位置 | 类型 | 必填 | 说明 |
|------|------|------|------|------|
| `hostId` | path | string | 是 | 已登记的 SSH 主机 id |

### 1.3 响应 200

```json
[
  {
    "id": "8c4f...-uuid",
    "hostId": "ab12...-uuid",
    "name": "nginx",
    "baseDir": "/opt/dockerApps/nginx",
    "composeFile": "docker-compose.yml",
    "note": null,
    "createdAt": 1735000000000,
    "updatedAt": 1735000000000
  }
]
```

---

## 2. 新增应用

### 2.1 基本信息
- **方法**：`POST`
- **路径**：`/api/docker/hosts/{hostId}/apps`
- **幂等**：否

### 2.2 请求 Body

```json
{
  "name": "nginx",
  "baseDir": "/opt/dockerApps/nginx",
  "composeFile": "docker-compose.yml",
  "note": "对外反向代理",
  "skipValidate": false
}
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `name` | string | 是 | - | 应用展示名 |
| `baseDir` | string | 是 | - | 绝对路径，长度 ≤ 512 |
| `composeFile` | string | 否 | `docker-compose.yml` | 相对 baseDir 的文件名 |
| `note` | string | 否 | null | 备注，长度 ≤ 1024 |
| `skipValidate` | boolean | 否 | false | true 时跳过 `docker compose config -q` 校验 |

### 2.3 响应 201
Body 同 §1.3 单条；触发 `(host_id, base_dir)` 重复返回 409 `{"error":"DUPLICATE"}`；compose 校验失败且 `skipValidate=false` 返回 422 `{"error":"COMPOSE_INVALID","message":"<stderr>"}`。

---

## 3. 修改应用

`PUT /api/docker/hosts/{hostId}/apps/{appId}`，Body 同 §2.2；返回更新后的 `DockerAppView`。

---

## 4. 删除应用

`DELETE /api/docker/hosts/{hostId}/apps/{appId}` → 204；不调用任何远端命令。

---

## 5. 目录扫描

### 5.1 基本信息
- **方法**：`POST`
- **路径**：`/api/docker/hosts/{hostId}/scan`

### 5.2 请求 Body

```json
{ "baseDir": "/opt/dockerApps", "maxDepth": 3 }
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `baseDir` | string | 是 | - | 扫描根，绝对路径 |
| `maxDepth` | int | 否 | 3 | `find -maxdepth`，范围 [1,5] |

### 5.3 响应 200

```json
{
  "items": [
    {
      "baseDir": "/opt/dockerApps/nginx",
      "composeFile": "docker-compose.yml",
      "name": "nginx",
      "registered": true,
      "existingAppId": "8c4f...-uuid"
    },
    {
      "baseDir": "/opt/dockerApps/redis",
      "composeFile": "compose.yml",
      "name": "redis",
      "registered": false,
      "existingAppId": null
    }
  ]
}
```

`name` 默认取 `baseDir` 最后一段。

---

## 6. 容器列表

### 6.1 基本信息
- **方法**：`GET`
- **路径**：`/api/docker/hosts/{hostId}/containers`

### 6.2 Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `appId` | string | 否 | - | 仅返回属于该应用的容器（按 compose project label 匹配） |
| `includeStopped` | boolean | 否 | true | false 时仅返回 running |
| `nocache` | boolean | 否 | false | true 时绕过 30s 服务端缓存，强制刷新远端 |

> 服务端对 `(hostId, appId, includeStopped)` 组合做 30s 内存缓存；本工具自身的写操作（容器动作 / compose 动作）会主动失效缓存；外部变更（别人 `docker run` 或主机 reboot）需要前端按「刷新」传 `nocache=true` 感知。

### 6.3 响应 200

```json
[
  {
    "id": "9f...",
    "shortId": "9f12abcd",
    "name": "nginx_web_1",
    "image": "nginx:1.27",
    "state": "running",
    "status": "Up 3 hours",
    "createdAt": 1735000000,
    "ports": "0.0.0.0:80->80/tcp",
    "composeProject": "nginx",
    "composeService": "web",
    "appId": "8c4f...-uuid"
  }
]
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 完整容器 ID |
| `state` | string | `running` / `exited` / `paused` / `created` / `restarting` |
| `status` | string | docker 原生 status 字符串 |
| `composeProject` | string\|null | 来自 label `com.docker.compose.project` |
| `appId` | string\|null | 根据 composeProject 与已登记 baseDir 反查 |

---

## 7. 容器单动作

### 7.1 基本信息
- **方法**：`POST`
- **路径**：`/api/docker/hosts/{hostId}/containers/{cid}/{action}`
- `action` ∈ §通用枚举 `containerAction`

### 7.2 请求
仅 path 参数；无 body。

### 7.3 响应

| 状态码 | 含义 | Body |
|--------|------|------|
| 204 | 命令 exitCode=0 | 无 |
| 400 | action 非白名单 | `{error:"BAD_ACTION"}` |
| 500 | 远端命令失败 | `{error:"REMOTE_FAIL","message":"<stderr>"}` |

---

## 8. 资源快照

### 8.1 基本信息
- **方法**：`GET`
- **路径**：`/api/docker/hosts/{hostId}/containers/stats`

### 8.2 Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `nocache` | boolean | 否 | false | true 绕过 30s 缓存 |

### 8.3 响应 200

```json
{
  "snapshotAt": 1735000000000,
  "items": [
    {
      "id": "9f...",
      "name": "nginx_web_1",
      "cpuPercent": 0.32,
      "memUsageBytes": 12582912,
      "memLimitBytes": 8589934592,
      "memPercent": 0.15,
      "netRxBytes": 102400,
      "netTxBytes": 51200,
      "blockReadBytes": 0,
      "blockWriteBytes": 4096
    }
  ]
}
```

底层执行 `docker stats --no-stream --format '{{json .}}'` 后解析。

---

## 9. compose 动作

### 9.1 基本信息
- **方法**：`POST`
- **路径**：`/api/docker/hosts/{hostId}/apps/{appId}/compose/{action}`
- `action` ∈ §通用枚举 `composeAction`

### 9.2 请求 Body（可选）

```json
{ "detach": true, "removeOrphans": false, "pullPolicy": "missing" }
```

| 字段 | 类型 | 必填 | 默认 | 适用动作 | 说明 |
|------|------|------|------|----------|------|
| `detach` | boolean | 否 | true | up | `-d` |
| `removeOrphans` | boolean | 否 | false | up / down | `--remove-orphans` |
| `pullPolicy` | string | 否 | `missing` | up | `always` / `missing` / `never` |

### 9.3 响应 200

```json
{ "exitCode": 0, "stdout": "...", "stderr": "...", "durationMs": 1820 }
```

失败保留 200 + `exitCode!=0`，前端按 exitCode 决定 toast 样式（与单容器动作的 500 设计不同：compose 长输出需要展示完整 stdout/stderr）。

---

## 10. 一次性日志 tail

### 10.1 基本信息
- **方法**：`GET`
- **路径**：`/api/docker/hosts/{hostId}/containers/{cid}/logs`

### 10.2 Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `tail` | int | 否 | 200 | 范围 [1, 5000] |
| `since` | string | 否 | - | docker logs `--since`（如 `10m`、`2024-01-01T00:00:00`） |
| `timestamps` | boolean | 否 | false | `--timestamps` |

### 10.3 响应 200

```json
{ "lines": ["2026-05-27 ... line 1", "..."], "truncated": false }
```

`truncated=true` 表示输出超过 1MB 上限被截断。

---

## 11. SSE 流式 follow

### 11.1 基本信息
- **方法**：`GET`
- **路径**：`/api/docker/hosts/{hostId}/containers/{cid}/logs/stream`
- **Accept**：`text/event-stream`

### 11.2 Query

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `tail` | int | 否 | 200 | 初始回放行数，范围 [0, 5000] |
| `since` | string | 否 | - | 同 §10.2 |
| `timestamps` | boolean | 否 | false | 同 §10.2 |

### 11.3 事件

| event | 频率 | data 含义 |
|-------|------|----------|
| `ready` | 1 次（开头） | `{"streamId":"..."}` 客户端记下用于关闭 |
| `log` | 每行 | base64 编码的 UTF-8 行（不含换行符） |
| `heartbeat` | 每 15s | `{"ts":1735000000000}` 防中间件断开 |
| `error` | 仅 1 次（失败时） | `{"message":"..."}` 随后服务端关闭 |
| `done` | 1 次（远端进程退出 / 客户端关闭） | `{"exitCode":0}` |

### 11.4 错误码

| HTTP | 含义 |
|------|------|
| 200 | SSE 建立成功（后续走事件） |
| 404 | host/container 不存在 |
| 503 | SSH 连接失败 |

---

## 12. 主动关闭日志流

### 12.1 基本信息
- **方法**：`DELETE`
- **路径**：`/api/docker/streams/{streamId}`

### 12.2 响应

| 状态码 | 含义 |
|--------|------|
| 204 | 成功（或流已不存在，也返回 204） |

---

## 13. 列应用目录下的配置文件

### 13.1 基本信息
- **方法**：`GET`
- **路径**：`/api/docker/hosts/{hostId}/apps/{appId}/files`

### 13.2 响应 200

```json
[
  { "path": "/opt/dockerApps/nginx/docker-compose.yml", "name": "docker-compose.yml", "sizeBytes": 1240, "modifiedAt": 1735000000000 },
  { "path": "/opt/dockerApps/nginx/.env",               "name": ".env",               "sizeBytes": 320,  "modifiedAt": 1735000000000 }
]
```

筛选规则（白名单）：`docker-compose.y[a]ml`、`compose.y[a]ml`、`*.env`、`.env*`、`*.conf`、`*.yml`、`*.yaml`、`*.json`。

---

## 14. 读单文件

### 14.1 基本信息
- **方法**：`GET`
- **路径**：`/api/docker/hosts/{hostId}/apps/{appId}/files/content`

### 14.2 Query

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `path` | string | 是 | 绝对路径；服务端用 realpath 校验必须在 baseDir 子树 |

### 14.3 响应 200

```json
{ "path": "/opt/dockerApps/nginx/docker-compose.yml", "content": "...", "sizeBytes": 1240, "modifiedAt": 1735000000000 }
```

### 14.4 错误

| HTTP | 含义 |
|------|------|
| 403 | 路径越权（不在 baseDir 子树下） |
| 404 | 文件不存在 |
| 413 | 文件大于 256KB |

---

## 15. 保存单文件

### 15.1 基本信息
- **方法**：`PUT`
- **路径**：`/api/docker/hosts/{hostId}/apps/{appId}/files/content`

### 15.2 请求 Body

```json
{ "path": "/opt/dockerApps/nginx/docker-compose.yml", "content": "version: '3.8'\n..." }
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `path` | string | 是 | 同 §14.2 |
| `content` | string | 是 | UTF-8；长度 ≤ 262144（256KB） |

### 15.3 响应 200

```json
{
  "path": "/opt/dockerApps/nginx/docker-compose.yml",
  "backupPath": "/opt/dockerApps/nginx/docker-compose.yml.bak.1735000000000",
  "sizeBytes": 1240,
  "modifiedAt": 1735000000000
}
```

### 15.4 错误

| HTTP | 含义 |
|------|------|
| 403 | 路径越权 |
| 413 | 内容超大 |
| 500 | 远端 `cp / tee / mv` 任一失败；保留备份 |
