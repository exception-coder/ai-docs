# java8gu GitHub 数据源（轻量）

> 模版：lightweight-template.md
> 最后更新：2026-05-24
> 范围：kai-toolbox / frontend / src/features/java8gu

## 背景

`java8gu` 模块原本通过 Node 构建脚本 `scripts/build-java8gu-index.mjs` 扫描本机 `D:\Users\zhang\IdeaProjects\job-interview-log\java8gu-速记版`，
产出 `public/java8gu/index.json` + `public/java8gu/q/{id}.md` 作为前端静态数据源。

切换为运行时直接从 GitHub 仓库 `exception-coder/JobInterviewLog` 的 `java8gu-速记版/` 目录拉取目录结构与文件，
以便不再依赖本机文件 + 构建脚本，跨机器可用、内容随上游仓库实时同步。

## 1. 代码入口

| 文件 | 角色 |
|---|---|
| [frontend/src/features/java8gu/data.ts](../../../../IdeaProjects/kai-toolbox/frontend/src/features/java8gu/data.ts) | 数据加载层（重写） |
| [frontend/src/features/java8gu/lib/analyze.ts](../../../../IdeaProjects/kai-toolbox/frontend/src/features/java8gu/lib/analyze.ts) | 浏览器端 markdown 元数据提取（新增，对齐 build 脚本 `analyze()`） |
| [frontend/src/features/java8gu/pages/Java8guHubPage.tsx](../../../../IdeaProjects/kai-toolbox/frontend/src/features/java8gu/pages/Java8guHubPage.tsx) | Hub 页加进度指示 |

构建脚本 `scripts/build-java8gu-index.mjs` 不再被运行时依赖，保留作为可选离线生成器（用户文档中不再要求执行）。

## 2. 接口契约（模块内 API）

```ts
export interface DataSourceConfig {
  owner: string
  repo: string
  branch: string
  dir: string        // 仓库根下子目录，留空 = 从仓库根扫
  token?: string     // 可选 GitHub PAT；填了把 Trees API 限流从 60/h 升到 5000/h
}

export interface LoadOptions {
  onProgress?: (done: number, total: number) => void
  forceRefresh?: boolean   // 用户点「刷新」时传 true，否则 cache-first
}

export function loadIndex(opts?: LoadProgress | LoadOptions): Promise<Java8guIndex>
export function loadMarkdown(id: string): Promise<string>
export function getDataSource(): DataSourceConfig
export function setDataSource(cfg: DataSourceConfig): void
export function findCategory(index, id): Java8guCategory | undefined
export function findQuestion(index, id): Java8guQuestion | undefined
export function viewCategory(index, id): CategoryView | undefined
```

数据源配置存 `localStorage['java8gu:source:v1']`；默认值 = `exception-coder/JobInterviewLog@main:java8gu-速记版`。
UI 入口：Hub 页右上「数据源」抽屉编辑（owner/repo/branch/dir/token）+ 「刷新」按钮强制重拉。

## 3. 核心流程

```mermaid
flowchart TD
    A([loadIndex 调用]) --> FORCE{forceRefresh?}
    FORCE -->|否 默认| CACHE{localStorage<br/>有该数据源缓存?}
    CACHE -->|是| RETURN([直接返回缓存索引<br/>不打 Trees API])
    CACHE -->|否| TREE
    FORCE -->|是 用户点刷新| TREE[GET /repos/owner/repo/git/trees/branch?recursive=1<br/>有 token 时带 Authorization Bearer]
    TREE -->|tree.sha| SHACHK{已缓存 sha == 新 sha?}
    SHACHK -->|是| RETURN
    SHACHK -->|否| FILTER[过滤 path 以 {dir}/ 开头<br/>且匹配 NN_xxx/NNNN_xxx.md]
    FILTER --> GROUP[按一级目录归类<br/>→ categories + questions 列表]
    GROUP --> FETCH[并发拉取 raw.githubusercontent.com<br/>限 16 路并发]
    FETCH -->|每完成一份| PROGRESS[onProgress done++ ]
    PROGRESS --> FETCH
    FETCH -->|全部完成| ANALYZE[逐份 analyzeMarkdown]
    ANALYZE --> ASSEMBLE[组装 Java8guIndex]
    ASSEMBLE --> SAVE[写 localStorage<br/>java8gu:index:v1:{owner/repo@branch:dir}]
    SAVE --> RETURN

    LOADMD([loadMarkdown id]) --> MEM{module Map 命中?}
    MEM -->|是| RETMD([返回缓存 markdown])
    MEM -->|否| RAW[GET raw.githubusercontent.com/{dir}/{sourceFile}]
    RAW --> RETMD
```

## 4. 关键规则

| 规则 | 说明 |
|------|------|
| **目录结构识别** | 同时尝试两种结构、嵌套优先：<br/>**(a) 嵌套**：`{dir}/NN_类目/NNNN_题目.md` —— 一级目录名 = `categoryId`，去掉 `NN_` 前缀作 `categoryLabel`<br/>**(b) 平铺**：`{dir}/NN_主题.md` —— 全部归入合成 category `__flat__`，category label = dir basename，每个文件 = 1 question；只有嵌套扫不到任何文件才回落平铺 |
| **question id** | 嵌套：4 位数字前缀（如 `0054`）；平铺：2 位数字前缀（如 `01`）。全库内唯一即可 |
| **markdown URL** | `https://raw.githubusercontent.com/{owner}/{repo}/{branch}/{encodeURI(sourceFile)}`；中文目录名/文件名必须 encodeURI 后再请求 |
| **并发** | 单批次并发 16，避免一次性打 1300 个连接被 raw CDN 节流 |
| **缓存键** | localStorage key = `java8gu:index:v1:{owner}/{repo}@{branch}:{dir}`，value = `{ sha, index }`；按数据源指纹隔离，切数据源不串 |
| **Cache-first** | 非 forceRefresh 场景，本地有索引就直接返回，**不打 Trees API**。仅 (a) 用户点「刷新」 (b) 切换数据源 (c) 本地没缓存 三种情况才打 Trees API |
| **Token 认证** | 配置里有 `token` 时 Trees API 请求带 `Authorization: Bearer <token>`，限流从匿名 60/h 升到 5000/h；raw.githubusercontent.com 不带 token（无需且增加泄露面） |
| **缓存大小** | 仅缓存 `index`（约 700KB），不缓存全部 markdown（避免超出 localStorage 配额） |
| **markdown 缓存** | module 级 `Map<string, Promise<string>>`，刷新页面即失效（设计选择：题目 markdown 平均 5KB，浏览器 HTTP 缓存足够） |
| **analyze 口径** | `lib/analyze.ts` 与 `scripts/build-java8gu-index.mjs` 的 `analyze()` 输出字段语义 1:1 对齐（title/tldr/chars/readMin/headings/codeCount/codeLangs/hasTable/hasImage/difficulty/difficultyScore） |

## 5. 失败行为

| 失败 | 处理 |
|------|------|
| Trees API 403 / 429（限流） | 抛错时附带剩余配额 + 重置时间 + 提示「填 token 可升到 5000/h」；正常情况下 cache-first 已经避免大部分限流 |
| Trees API 404（仓库/分支不存在） | 抛错明示「请检查数据源配置」 |
| Trees API 5xx / 网络断 | 抛错，Hub 页错误条显示原始 HTTP code 与建议 |
| 单个 markdown 拉取失败 | 标记该题为 `analyze` 默认值（难度=1、tldr=空），继续其他文件；最终成功率 < 90% 时整体置失败 |
| localStorage 写入失败（quota exceeded） | 静默忽略，不影响本次内存索引使用 |
| localStorage JSON 解析失败 | 删除该 key，按未命中处理 |

## 6. 不做（边界）

- 不做后端代理：所有请求直接从浏览器打 GitHub，**用户在公司网络下需自行确保可达 github.com 与 raw.githubusercontent.com**
- 不做 OAuth：匿名访问够用（公共仓库，60 req/h Trees + 5000 req/h raw 对单用户足够）
- 不做增量同步：sha 不一致即全量重拉，简单可靠
- 不删除 `scripts/build-java8gu-index.mjs` 与 `public/java8gu/`：保留作为离线 fallback 与历史归档；运行时不再读取

## 7. 升级触发条件（命中则升级到完整模版）

- 引入后端代理 / OAuth → 跨模块、新对外契约 → 完整-技术
- 接入多仓库 / 用户可配置仓库 → 状态机 + UI 配置 → 完整-业务
- 与其他工具共用 GitHub 拉取层（抽公共 lib） → 跨模块 → 完整-技术
