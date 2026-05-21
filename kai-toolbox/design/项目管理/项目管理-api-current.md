# 项目管理 - 接口契约

> 最后更新：2026-05-11
> 本文档承载本工具对外的接口契约：2 个 HTTP 接口。

---

## 1. 端点总览

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/api/projects` | 拉取扫描结果（含 5 秒后端缓存） |
| POST | `/api/projects/open` | 在系统文件管理器中打开指定项目目录 |

| 项 | 值 |
|---|---|
| 鉴权 | 无（与项目整体一致） |
| Content-Type | `application/json; charset=UTF-8` |
| 错误响应 | 走全局 `GlobalExceptionHandler`，正文形如 `{"code":"...","message":"..."}` |

---

## 2. GET /api/projects

### 2.1 请求

无入参。

### 2.2 响应

```json
{
  "root": "D:\\Users\\zhang\\IdeaProjects",
  "rootExists": true,
  "scannedAt": "2026-05-11T10:23:45+08:00",
  "items": [
    {
      "name": "kai-toolbox",
      "path": "D:\\Users\\zhang\\IdeaProjects\\kai-toolbox",
      "type": "maven",
      "branch": "main",
      "lastModified": "2026-05-10T22:11:08+08:00"
    },
    {
      "name": "team-standards",
      "path": "D:\\Users\\zhang\\IdeaProjects\\team-standards",
      "type": "node",
      "branch": "main",
      "lastModified": "2026-05-09T18:42:01+08:00"
    },
    {
      "name": "docs",
      "path": "D:\\Users\\zhang\\IdeaProjects\\docs",
      "type": "other",
      "branch": null,
      "lastModified": "2026-04-28T10:15:00+08:00"
    }
  ]
}
```

#### 2.2.1 顶层字段

| 字段 | 类型 | 说明 |
|---|---|---|
| root | string | 当前生效的扫描根（来自配置，原样回显，便于前端展示） |
| rootExists | bool | 根目录是否存在；为 false 时 items 为空数组 |
| scannedAt | string (ISO-8601) | 本次扫描完成时间；命中缓存时为缓存生成时间 |
| items | ProjectInfo[] | 项目列表，按 lastModified 倒序 |

#### 2.2.2 ProjectInfo 字段

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| name | string | 是 | 项目目录名（一级子目录名） |
| path | string | 是 | 项目绝对路径（Windows 反斜杠原样保留） |
| type | enum | 是 | 见 2.2.3 |
| branch | string \| null | 否 | 仅 type 含 `.git` 时尝试读取；读取失败为 null |
| lastModified | string (ISO-8601) | 是 | `Files.getLastModifiedTime(path)` |

#### 2.2.3 ProjectType 枚举

| 值 | 触发条件（按优先级，首命中） |
|---|---|
| `flutter` | 包含 `pubspec.yaml` |
| `maven` | 包含 `pom.xml` |
| `gradle` | 包含 `build.gradle` 或 `build.gradle.kts` 或 `settings.gradle` |
| `node` | 包含 `package.json` |
| `python` | 包含 `pyproject.toml` 或 `requirements.txt` |
| `git` | 仅含 `.git` 但无以上签名 |
| `other` | 都未命中 |

> `.git` 在前 6 类下独立用于读取 `branch` 字段，不影响 type 判定。

---

## 3. POST /api/projects/open

### 3.1 用途

通过 `java.awt.Desktop.open()` 在系统文件管理器（Windows = Explorer）中打开指定路径。仅打开**已存在的目录**；不接受文件路径。

### 3.2 请求

```json
{
  "path": "D:\\Users\\zhang\\IdeaProjects\\kai-toolbox"
}
```

| 字段 | 类型 | 必填 | 校验 |
|---|---|---|---|
| path | string | 是 | 必须是已存在目录；必须以 `toolbox.projects.root` 为前缀（防越权） |

### 3.3 响应

成功：`204 No Content`。

失败：

```json
{
  "code": "INVALID_PATH",
  "message": "path is outside projects root"
}
```

| `code` | HTTP | 含义 |
|---|---|---|
| `INVALID_PATH` | 400 | path 非法 / 不存在 / 不在 root 内 |
| `HEADLESS_ENVIRONMENT` | 503 | 服务端无桌面会话，无法打开文件管理器 |
| `INTERNAL_ERROR` | 500 | 兜底 |

---

## 4. 与前端跳转的协作（说明性，不构成新接口）

本模块**不**新建跳转专用接口。前端点击卡片后直接走 `react-router` 跳转：

```text
/tools/webterm?cwd=<URL-encoded 绝对路径>&autorun=claude
```

Web 终端模块的 `WebTermPage` 解析 query：
- `cwd`：拼入首发 `open` 消息的 `cwd` 字段（既有契约支持，无需改）
- `autorun`：白名单仅 `claude`；收到 `ready` 后由前端追发 `{type:"input", data:"claude\r\n"}`

> **Web 终端后端契约不变。** 见 `Web终端/Web终端-api-current.md` 第 2.1 节。

---

## 5. 配置项

`application.yml`：

| 键 | 默认 | 说明 |
|---|---|---|
| `toolbox.projects.root` | `D:\Users\zhang\IdeaProjects` | 扫描根目录绝对路径 |
| `toolbox.projects.cache-ttl-seconds` | `5` | 后端列表缓存 TTL |
| `toolbox.projects.hidden-prefixes` | `[".", "_"]` | 一级目录名以这些前缀开头时跳过 |
