# GitHub 文档浏览器 — 接口文档

> 配套 `GitHub文档浏览器-current.md` 技术方案使用，互不重复。
> 字段级 detail 在本文档维护；设计决策、流程、风险在配套方案文档维护。
> 路径前缀：`/api/doc-viewer`

---

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|--------------|
| v1 | 2026-05-08 | AI 草稿 | 初始版本：6 个 REST 接口 |

---

## 接口清单

| # | 方法 | 路径 | 用途 |
|---|------|------|------|
| 1 | POST | /api/doc-viewer/sources | 添加文档源（粘贴 GitHub URL + 可选 PAT） |
| 2 | GET | /api/doc-viewer/sources | 列出所有已收藏文档源 |
| 3 | DELETE | /api/doc-viewer/sources/{id} | 删除文档源（连带缓存） |
| 4 | POST | /api/doc-viewer/sources/{id}/refresh | 手动刷新该源的树（ETag 校验） |
| 5 | GET | /api/doc-viewer/sources/{id}/tree | 获取整棵目录树 |
| 6 | GET | /api/doc-viewer/sources/{id}/file | 获取单个文件内容（按 path） |

---

## 1. 添加文档源

### 1.1 基本信息

- **方法**：`POST`
- **路径**：`/api/doc-viewer/sources`
- **用途**：粘贴 GitHub URL + 可选 PAT，解析后保存并首次拉取整棵树
- **认证**：不需要（本地工具）
- **幂等**：是。(owner, repo, ref, subPath) 联合唯一，重复 POST 返回已有 source

### 1.2 请求

#### Headers

| Name | 必填 | 说明 |
|------|------|------|
| `Content-Type` | 是 | `application/json` |

#### Body

```json
{
  "url": "https://github.com/torvalds/linux/tree/master/Documentation",
  "pat": null,
  "alias": "Linux 内核文档"
}
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `url` | string | 是 | - | 任意 GitHub URL，支持 `/tree/`、`/blob/`、根仓库三种形态 |
| `pat` | string | 否 | null | GitHub Personal Access Token；私有仓库必填；公库可省 |
| `alias` | string | 否 | 自动生成 | 卡片显示名；省略时取 `{repo}/{subPath}` |

### 1.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": {
    "id": "src_a1b2c3",
    "owner": "torvalds",
    "repo": "linux",
    "ref": "master",
    "subPath": "Documentation",
    "alias": "Linux 内核文档",
    "hasPat": false,
    "treeETag": "W/\"abc123\"",
    "rateLimitUntil": null,
    "lastRefreshedAt": 1715155200000,
    "createdAt": 1715155200000
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 文档源 ID（`src_` 前缀 + 短 hash） |
| `owner` | string | GitHub owner |
| `repo` | string | GitHub repo |
| `ref` | string | 分支或 tag |
| `subPath` | string | 仓库内子路径（无则空串） |
| `alias` | string | 卡片显示名 |
| `hasPat` | boolean | 是否设置 PAT，**永不回明文** |
| `treeETag` | string \| null | 当前缓存树的 ETag，前端无需关心，纯内部状态展示 |
| `rateLimitUntil` | integer \| null | 限流冷却结束时间戳（毫秒）；null 表示未限流 |
| `lastRefreshedAt` | integer | 最后一次成功刷新时间戳（毫秒） |
| `createdAt` | integer | 创建时间戳（毫秒） |

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `INVALID_GITHUB_URL` | 400 | URL 不可解析为 (owner, repo, ref) |
| `REPO_NOT_FOUND` | 404 | 仓库不存在或无权限 |
| `REPO_FORBIDDEN` | 403 | 私库 + 未提供 PAT 或 PAT 无效 |
| `TREE_TOO_LARGE` | 400 | 整树节点 > 100000，超出 GitHub Trees API 上限 |
| `UPSTREAM_UNAVAILABLE` | 502 | GitHub API 不可达（网络异常） |
| `RATE_LIMITED` | 429 | GitHub 限流，无可用配额且无缓存可用 |

### 1.4 示例

```http
POST /api/doc-viewer/sources HTTP/1.1
Content-Type: application/json

{
  "url": "https://github.com/torvalds/linux/tree/master/Documentation",
  "alias": "Linux 内核文档"
}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"code":0,"data":{"id":"src_a1b2c3","owner":"torvalds","repo":"linux","ref":"master","subPath":"Documentation","alias":"Linux 内核文档","hasPat":false,"treeETag":"W/\"abc123\"","rateLimitUntil":null,"lastRefreshedAt":1715155200000,"createdAt":1715155200000}}
```

---

## 2. 列出文档源

### 2.1 基本信息

- **方法**：`GET`
- **路径**：`/api/doc-viewer/sources`
- **用途**：拉取所有已收藏文档源，用于首页书架卡片
- **认证**：不需要
- **幂等**：是

### 2.2 请求

无参数。

### 2.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": [
    {"id": "src_a1b2c3", "owner": "torvalds", "repo": "linux", "ref": "master", "subPath": "Documentation", "alias": "Linux 内核文档", "hasPat": false, "lastRefreshedAt": 1715155200000, "rateLimitUntil": null}
  ]
}
```

字段同接口 1，省略内部字段（treeETag、createdAt）以减小体积。

### 2.4 示例

```http
GET /api/doc-viewer/sources HTTP/1.1
```

---

## 3. 删除文档源

### 3.1 基本信息

- **方法**：`DELETE`
- **路径**：`/api/doc-viewer/sources/{id}`
- **用途**：删除文档源记录，并级联清理 doc_tree_cache；doc_file_cache 不立即清理（按 sha 唯一，可能被其他源复用），由后续 LRU 任务回收
- **认证**：不需要
- **幂等**：是

### 3.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 文档源 ID |

### 3.3 响应

#### 成功（204）

无 body。

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `SOURCE_NOT_FOUND` | 404 | id 不存在 |

---

## 4. 手动刷新文档源

### 4.1 基本信息

- **方法**：`POST`
- **路径**：`/api/doc-viewer/sources/{id}/refresh`
- **用途**：发起一次树刷新；走 ETag 校验，304 沿用缓存，200 替换缓存，403 进入冷却期沿用缓存
- **认证**：不需要
- **幂等**：是（结果由远端状态决定，多次调用结果一致）

### 4.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 文档源 ID |

### 4.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": {
    "id": "src_a1b2c3",
    "outcome": "UPDATED",
    "treeETag": "W/\"def456\"",
    "lastRefreshedAt": 1715158800000,
    "rateLimitUntil": null,
    "rateLimited": false
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `outcome` | string enum | `NOT_MODIFIED` / `UPDATED` / `RATE_LIMITED` / `COOLDOWN`（冷却期内未发起请求） |
| `treeETag` | string | 当前缓存的 ETag |
| `lastRefreshedAt` | integer | 最后一次成功刷新时间戳（毫秒） |
| `rateLimitUntil` | integer \| null | 限流冷却结束时间戳；null 表示未限流 |
| `rateLimited` | boolean | 本次刷新是否处于限流降级状态（true 时返回的是缓存） |

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `SOURCE_NOT_FOUND` | 404 | id 不存在 |
| `UPSTREAM_UNAVAILABLE` | 502 | GitHub 不可达 |

---

## 5. 获取目录树

### 5.1 基本信息

- **方法**：`GET`
- **路径**：`/api/doc-viewer/sources/{id}/tree`
- **用途**：获取该源的整棵目录树（扁平列表，前端自行 nest）
- **认证**：不需要
- **幂等**：是

### 5.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 文档源 ID |

### 5.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": {
    "sourceId": "src_a1b2c3",
    "ref": "master",
    "refSha": "abc123def456",
    "rateLimited": false,
    "nodes": [
      {"path": "INDEX.md", "name": "INDEX.md", "kind": "BLOB", "sha": "blob1", "size": 1234, "parentPath": "", "depth": 0},
      {"path": "design", "name": "design", "kind": "TREE", "sha": "tree1", "size": null, "parentPath": "", "depth": 0},
      {"path": "design/architecture.md", "name": "architecture.md", "kind": "BLOB", "sha": "blob2", "size": 5678, "parentPath": "design", "depth": 1}
    ]
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `sourceId` | string | 回显 |
| `ref` | string | 分支/tag 名 |
| `refSha` | string | 当前 ref 解析到的 commit sha |
| `rateLimited` | boolean | 是否处于限流降级（true 表示数据可能不是最新） |
| `nodes` | array | 扁平节点列表，按 path 字典序 |
| `nodes[].path` | string | 相对仓库根（含 subPath）的完整 path |
| `nodes[].name` | string | basename |
| `nodes[].kind` | string enum | `BLOB`（文件）/ `TREE`（目录）/ `BINARY`（非 md/文本的 blob） |
| `nodes[].sha` | string | git blob/tree sha |
| `nodes[].size` | integer \| null | 文件字节数；目录为 null |
| `nodes[].parentPath` | string | 父目录 path；根目录子项为空串 |
| `nodes[].depth` | integer | 树深度，0 表示根 |

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `SOURCE_NOT_FOUND` | 404 | id 不存在 |

---

## 6. 获取单个文件内容

### 6.1 基本信息

- **方法**：`GET`
- **路径**：`/api/doc-viewer/sources/{id}/file`
- **用途**：按 path 拉取该源中某个文件的正文；缓存优先
- **认证**：不需要
- **幂等**：是

### 6.2 请求

#### Path

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 文档源 ID |

#### Query

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `path` | string | 是 | 仓库内文件 path（含 subPath，与 tree 接口的 `nodes[].path` 一致） |

### 6.3 响应

#### 成功（200）

```json
{
  "code": 0,
  "data": {
    "sourceId": "src_a1b2c3",
    "path": "design/architecture.md",
    "sha": "blob2",
    "kind": "BLOB",
    "size": 5678,
    "content": "# Architecture\n\n...",
    "rawBaseUrl": "https://raw.githubusercontent.com/torvalds/linux/abc123def456/"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `sourceId` | string | 回显 |
| `path` | string | 回显 |
| `sha` | string | 文件 blob sha |
| `kind` | string enum | `BLOB` / `BINARY`；`BINARY` 时 content 为 null |
| `size` | integer | 文件字节数。前端按此决定渐进渲染策略（< 500KB 直接渲染 / 500KB-2MB 显骨架 / > 2MB 默认走原始文本） |
| `content` | string \| null | 文件正文；`BINARY` 时为 null。**不因 size 截断**——后端永远返回完整内容 |
| `rawBaseUrl` | string | 该 ref 对应的 raw URL 基址，前端按此重写相对图片/链接 |

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `SOURCE_NOT_FOUND` | 404 | id 不存在 |
| `FILE_NOT_IN_TREE` | 404 | path 不在 doc_tree_cache 中（缓存可能过期，提示用户刷新） |
| `UPSTREAM_UNAVAILABLE` | 502 | GitHub raw 不可达且本地无缓存 |
| `RATE_LIMITED` | 429 | 限流且本地无缓存 |

### 6.4 示例

```http
GET /api/doc-viewer/sources/src_a1b2c3/file?path=design%2Farchitecture.md HTTP/1.1
```
