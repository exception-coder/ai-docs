# GitHub 文档浏览器 — 编码摘要

> 配套 `GitHub文档浏览器-current.md` 技术方案 + `GitHub文档浏览器-api-current.md` 接口契约。
> 本文档只回答「每个方法怎么写」；契约 detail / 设计取舍 / 风险描述均见配套文档。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-05-08 | 初始版本：5 个后端类 + 6 个前端组件方法签名与实现要点 |

---

## 1. 核心业务规则

- 文档源 (owner, repo, ref, subPath) 联合唯一；重复 POST 返回已有 sourceId
- PAT 与 source 同行存储；返回前端时只回 `hasPat: true`，永不回写明文
- 目录优先索引：点目录时按 `INDEX.md` → `README.md` → `00_index.md` 顺序优先呈现
- 相对图片 `![](./x.png)` 重写为 `{rawBaseUrl}{currentDir}/x.png`；相对 md 链接重写为应用内路由
- 后端**不**对文件大小做硬截断；前端按 size 选渐进渲染策略
- 二进制文件 (`kind=BINARY`) Service 返 `content=null`，前端只展示文件名/大小
- 自动刷新：打开收藏源时若距上次成功刷新 > 1 小时则触发一次 refresh，独立失败不阻塞读
- 限流：`X-RateLimit-Remaining ≤ 5` 时 Service 主动 short-circuit，写 `rate_limit_until` 并返回缓存

---

## 2. 接口入口指针

| 接口 | 实现类 # 方法 |
|------|-------------|
| `POST /api/doc-viewer/sources` | `com.exceptioncoder.toolbox.docviewer.api.DocViewerController#createSource` |
| `GET /api/doc-viewer/sources` | `com.exceptioncoder.toolbox.docviewer.api.DocViewerController#listSources` |
| `DELETE /api/doc-viewer/sources/{id}` | `com.exceptioncoder.toolbox.docviewer.api.DocViewerController#deleteSource` |
| `POST /api/doc-viewer/sources/{id}/refresh` | `com.exceptioncoder.toolbox.docviewer.api.DocViewerController#refreshSource` |
| `GET /api/doc-viewer/sources/{id}/tree` | `com.exceptioncoder.toolbox.docviewer.api.DocViewerController#getTree` |
| `GET /api/doc-viewer/sources/{id}/file` | `com.exceptioncoder.toolbox.docviewer.api.DocViewerController#getFile` |

---

## 3. 涉及类清单（全路径）

### 3.1 后端 — Java

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.toolbox.docviewer.api.DocViewerController` | 新建 | HTTP 入口，6 个方法 |
| `com.exceptioncoder.toolbox.docviewer.api.dto.SourceDTO` | 新建 | 文档源出参（不含 PAT 明文） |
| `com.exceptioncoder.toolbox.docviewer.api.dto.CreateSourceRequest` | 新建 | 创建文档源入参 |
| `com.exceptioncoder.toolbox.docviewer.api.dto.TreeNodeDTO` | 新建 | 树节点 |
| `com.exceptioncoder.toolbox.docviewer.api.dto.FileDTO` | 新建 | 文件正文出参 |
| `com.exceptioncoder.toolbox.docviewer.api.dto.RefreshOutcomeDTO` | 新建 | 刷新结果 |
| `com.exceptioncoder.toolbox.docviewer.service.DocViewerService` | 新建 | 编排：缓存优先 + ETag + 限流降级 |
| `com.exceptioncoder.toolbox.docviewer.infra.GitHubClient` | 新建 | GitHub REST + raw HTTP 客户端 |
| `com.exceptioncoder.toolbox.docviewer.infra.GitHubUrlParser` | 新建 | URL → owner/repo/ref/subPath（纯函数） |
| `com.exceptioncoder.toolbox.docviewer.infra.dto.RefSha` | 新建 | 内部 record，承载 commit sha + ETag |
| `com.exceptioncoder.toolbox.docviewer.infra.dto.TreeFetchResult` | 新建 | 内部 record，承载 nodes + ETag + rateLimitReset |
| `com.exceptioncoder.toolbox.docviewer.repository.DocCacheRepository` | 新建 | SQLite 仓储 |
| `com.exceptioncoder.toolbox.docviewer.repository.entity.DocSource` | 新建 | source 行实体 |
| `com.exceptioncoder.toolbox.docviewer.repository.entity.DocTreeNode` | 新建 | tree node 行实体 |
| `com.exceptioncoder.toolbox.docviewer.repository.entity.DocFileCache` | 新建 | file 行实体 |
| `com.exceptioncoder.toolbox.docviewer.exception.DocViewerException` | 新建 | 领域异常 + 错误码 enum |
| `com.exceptioncoder.toolbox.docviewer.exception.DocViewerErrorCode` | 新建 | enum：INVALID_GITHUB_URL / REPO_NOT_FOUND / REPO_FORBIDDEN / TREE_TOO_LARGE / RATE_LIMITED / UPSTREAM_UNAVAILABLE / SOURCE_NOT_FOUND / FILE_NOT_IN_TREE |
| `com.exceptioncoder.toolbox.docviewer.config.DocViewerToolDescriptor` | 新建 | `@Component` 实现 `ToolDescriptor`，注册到 `ToolRegistry` |

### 3.2 后端 — 资源

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `tools/tool-doc-viewer/pom.xml` | 新建 | Maven 模块声明，依赖 toolbox-common |
| `tools/tool-doc-viewer/src/main/resources/db/doc-viewer-schema.sql` | 新建 | 三张表 + 索引 DDL（全部 `IF NOT EXISTS`） |
| `pom.xml`（root） | 修改 | `<modules>` 加 `tools/tool-doc-viewer` |
| `toolbox-starter/pom.xml` | 修改 | `<dependency>` 加 tool-doc-viewer |

### 3.3 前端

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `frontend/src/features/doc-viewer/index.tsx` | 新建 | FeatureManifest 导出（icon: BookOpen, group: 学习/参考） |
| `frontend/src/features/doc-viewer/types.ts` | 新建 | TS 类型镜像后端 DTO |
| `frontend/src/features/doc-viewer/api/docViewer.ts` | 新建 | TanStack Query hooks |
| `frontend/src/features/doc-viewer/pages/DocViewerHome.tsx` | 新建 | 文档源书架卡片 |
| `frontend/src/features/doc-viewer/pages/DocViewerPage.tsx` | 新建 | 三栏布局壳 + 路由参数解析 |
| `frontend/src/features/doc-viewer/components/FileTree.tsx` | 新建 | 目录树 |
| `frontend/src/features/doc-viewer/components/MarkdownView.tsx` | 新建 | react-markdown 集成 |
| `frontend/src/features/doc-viewer/components/RawTextView.tsx` | 新建 | `<pre>` 兜底视图 |
| `frontend/src/features/doc-viewer/components/TocPanel.tsx` | 新建 | 从渲染产物提取 H2/H3 |
| `frontend/src/features/doc-viewer/components/AddSourceDialog.tsx` | 新建 | 添加文档源对话框 |
| `frontend/src/features/doc-viewer/lib/parseGitHubUrl.ts` | 新建 | 前端 URL 解析（仅用于即时校验提示，最终以后端为准） |
| `frontend/src/features/doc-viewer/lib/rewriteRelativeLinks.ts` | 新建 | rehype 插件：相对图片/链接重写 |
| `frontend/src/features/doc-viewer/lib/mermaidRenderer.tsx` | 新建 | mermaid 懒加载渲染组件 |
| `frontend/src/features/doc-viewer/lib/sizeStrategy.ts` | 新建 | 按 size 决定渲染策略（< 500KB 直接 / 500KB-2MB 骨架 / > 2MB raw） |
| `frontend/package.json` | 修改 | 新增 `react-markdown` `remark-gfm` `rehype-highlight` `mermaid` `highlight.js` |

### 关键方法签名与职责

```text
// === Controller ===
DocViewerController#createSource(CreateSourceRequest req): SourceDTO
  — 调 service.createOrGetSource，返回不含 PAT 明文的 DTO

DocViewerController#listSources(): List<SourceDTO>
DocViewerController#deleteSource(String id): ResponseEntity<Void>  // 204
DocViewerController#refreshSource(String id): RefreshOutcomeDTO
DocViewerController#getTree(String id): TreeResponseDTO
DocViewerController#getFile(String id, String path): FileDTO

// === Service ===
DocViewerService#createOrGetSource(String url, String pat, String alias): SourceDTO
  — 流程：parseUrl → 查 (owner,repo,ref,subPath) 唯一约束 → 命中则返已有；
    否则 getRefSha → getTree → upsertSource + replaceTreeCache → 返新 source

DocViewerService#listSources(): List<SourceDTO>
DocViewerService#deleteSource(String sourceId): void
  — 删 doc_source 行 + 删 doc_tree_cache 同 sourceId 行；doc_file_cache 不删（可能复用）

DocViewerService#refreshTree(String sourceId): RefreshOutcomeDTO
  — 流程：检查 rate_limit_until → 若在冷却期返 COOLDOWN；
    否则 githubClient.getTreeIfModified(refSha, etag) →
      304 返 NOT_MODIFIED；
      200 → replaceTreeCache + updateETag，返 UPDATED；
      403 RATE_LIMITED → 写 rate_limit_until，返 RATE_LIMITED

DocViewerService#getTree(String sourceId): TreeResponseDTO
  — 直接读 doc_tree_cache + source；附带 rateLimited 标记

DocViewerService#getFile(String sourceId, String path): FileDTO
  — 流程：lookupTreeNode(sourceId, path) → 拿 sha →
    lookupFile(sha) 命中则返；
    未命中 → githubClient.fetchRaw(owner,repo,refSha,path) → upsertFile(sha, content) → 返

// 同 sourceId 串行化用 ConcurrentHashMap<String, Object> locks; synchronized(locks.computeIfAbsent(id, k -> new Object()))

// === GitHubClient ===
GitHubClient#parseUrl(String url): GitHubCoord
  — 委托 GitHubUrlParser.parse；非法 URL 抛 INVALID_GITHUB_URL

GitHubClient#getRefSha(String owner, String repo, String ref, String pat): RefSha
  — GET https://api.github.com/repos/{owner}/{repo}/branches/{ref}
    Authorization: token {pat} 仅 pat 非空时附加
    404 → REPO_NOT_FOUND；403 → REPO_FORBIDDEN（含 message 区分私库 vs 限流）

GitHubClient#getTree(String owner, String repo, String sha, String pat): TreeFetchResult
  — GET https://api.github.com/repos/{owner}/{repo}/git/trees/{sha}?recursive=1
    truncated=true → TREE_TOO_LARGE
    返 nodes(path,sha,type=blob|tree,size) + ETag

GitHubClient#getTreeIfModified(String owner,String repo,String sha,String pat,String etag): TreeFetchResult | NotModified | RateLimited
  — 同上 + If-None-Match: {etag}
    304 → NotModified；403 + X-RateLimit-Remaining=0 → RateLimited(reset)

GitHubClient#fetchRaw(String owner,String repo,String refSha,String path,String pat): String
  — GET https://raw.githubusercontent.com/{owner}/{repo}/{refSha}/{path}
    UTF-8 解码返 content；二进制（非 text/* 且非可解码 UTF-8）抛 BINARY 标记

// === GitHubUrlParser（纯函数） ===
GitHubUrlParser#parse(String url): GitHubCoord
  — 接受三种形态：
    https://github.com/{o}/{r}                       → ref=default, subPath=""
    https://github.com/{o}/{r}/tree/{ref}/{subPath}  → 含子路径目录形态
    https://github.com/{o}/{r}/blob/{ref}/{path}     → 单文件形态，subPath=父目录，autoFocus=path

// === DocCacheRepository ===
DocCacheRepository#findSourceByCoord(owner, repo, ref, subPath): Optional<DocSource>
DocCacheRepository#findSourceById(id): Optional<DocSource>
DocCacheRepository#listAllSources(): List<DocSource>
DocCacheRepository#upsertSource(DocSource): void
DocCacheRepository#deleteSource(id): void
DocCacheRepository#updateSourceETagAndTimestamp(id, etag, lastRefreshedAt): void
DocCacheRepository#updateSourceRateLimitUntil(id, until): void

DocCacheRepository#replaceTreeCache(sourceId, List<DocTreeNode>): void
  — 事务：DELETE FROM doc_tree_cache WHERE source_id=?; batch INSERT

DocCacheRepository#listTreeNodes(sourceId): List<DocTreeNode>
DocCacheRepository#findTreeNode(sourceId, path): Optional<DocTreeNode>

DocCacheRepository#findFileBySha(sha): Optional<DocFileCache>
DocCacheRepository#upsertFile(DocFileCache): void
```

```typescript
// === 前端 ===

// api/docViewer.ts —— TanStack Query hooks
useSourcesQuery(): UseQueryResult<SourceDTO[]>
useCreateSourceMutation(): UseMutationResult<SourceDTO, Error, CreateSourceRequest>
useDeleteSourceMutation(): UseMutationResult<void, Error, string>
useRefreshSourceMutation(): UseMutationResult<RefreshOutcomeDTO, Error, string>
useTreeQuery(sourceId: string): UseQueryResult<TreeResponseDTO>
useFileQuery(sourceId: string, path: string | null): UseQueryResult<FileDTO>
  — enabled: path != null; staleTime: Infinity（文件按 sha 不变）

// pages/DocViewerPage.tsx
DocViewerPage()  // 路由 /tools/doc-viewer/:sourceId/*path
  — 解析路由 → useTreeQuery + useFileQuery
  — 状态：currentPath, viewMode (markdown | raw)
  — 当 path 指向目录或末路径 → 自动选 INDEX/README/00_index 之一作 currentFilePath

// components/FileTree.tsx
FileTree({nodes, currentPath, onSelect})
  — useMemo flat→nested 一次
  — 折叠状态用 useState<Set<string>>
  — 类似 INDEX/README/00_index 节点高亮

// components/MarkdownView.tsx
MarkdownView({content, rawBaseUrl, currentDir, size, viewMode, onToggleViewMode})
  — viewMode='raw' 渲染 RawTextView
  — viewMode='markdown' 渲染 ReactMarkdown，plugins:
    [remarkGfm, [rehypeHighlight], rehypeRewriteRelativeLinks({rawBaseUrl, currentDir, sourceId})]
    components: { code: CodeBlock(自动识别 mermaid 走 MermaidRenderer) }
  — size > 500KB 显骨架 + 异步 setTimeout(0) 渲染避免主线程阻塞

// lib/sizeStrategy.ts
chooseInitialViewMode(size: number): 'markdown' | 'raw'
  — size > 2*1024*1024 → 'raw'，否则 'markdown'
showSkeletonHint(size: number): boolean
  — size > 500*1024
```

---

## 4. 数据结构

### doc-viewer-schema.sql（完整 DDL）

```sql
CREATE TABLE IF NOT EXISTS doc_source (
    id                 TEXT PRIMARY KEY,
    owner              TEXT NOT NULL,
    repo               TEXT NOT NULL,
    ref_name           TEXT NOT NULL,
    sub_path           TEXT NOT NULL DEFAULT '',
    ref_sha            TEXT NOT NULL,
    alias              TEXT NOT NULL,
    pat                TEXT,
    tree_etag          TEXT,
    rate_limit_until   INTEGER,
    last_refreshed_at  INTEGER NOT NULL,
    created_at         INTEGER NOT NULL
);
CREATE UNIQUE INDEX IF NOT EXISTS uk_doc_source_coord
    ON doc_source(owner, repo, ref_name, sub_path);

CREATE TABLE IF NOT EXISTS doc_tree_cache (
    source_id     TEXT NOT NULL,
    path          TEXT NOT NULL,
    name          TEXT NOT NULL,
    kind          TEXT NOT NULL,        -- BLOB | TREE | BINARY
    sha           TEXT NOT NULL,
    size          INTEGER,
    parent_path   TEXT NOT NULL DEFAULT '',
    depth         INTEGER NOT NULL,
    PRIMARY KEY (source_id, path)
);
CREATE INDEX IF NOT EXISTS idx_tree_parent
    ON doc_tree_cache(source_id, parent_path);

CREATE TABLE IF NOT EXISTS doc_file_cache (
    sha          TEXT PRIMARY KEY,
    kind         TEXT NOT NULL,         -- BLOB | BINARY
    size         INTEGER NOT NULL,
    content      TEXT,                  -- BINARY 时为 NULL
    cached_at    INTEGER NOT NULL
);
```

### 关键 DTO 字段

```java
// CreateSourceRequest
String url;          // 必填，GitHub URL
String pat;          // 可选
String alias;        // 可选，省略时取 repo/subPath

// SourceDTO（不含 PAT 明文）
String id, owner, repo, ref, subPath, alias;
boolean hasPat;
String treeETag;            // 内部状态
Long rateLimitUntil;        // 毫秒时间戳，可空
long lastRefreshedAt, createdAt;

// FileDTO
String sourceId, path, sha;
String kind;                // BLOB | BINARY
long size;
String content;             // BINARY 时为 null；不因 size 截断
String rawBaseUrl;          // raw URL 基址，前端按此重写相对图片
```

---

## 5. 重要约束与边界

- **幂等键**：(owner, repo, ref, subPath) 联合唯一；createSource 重复请求返已有 source
- **并发控制**：同 sourceId 的 refreshTree / getFile 串行化（Service 内 `ConcurrentHashMap<String, Object>` + `synchronized`），避免重复回源
- **事务范围**：
  - `replaceTreeCache` 是单事务：DELETE 旧 + batch INSERT 新
  - 其他 CRUD 单条不走显式事务
- **不处理的场景**：
  - 不支持非 GitHub 仓库
  - 不支持 markdown 编辑（只读）
  - 不实现 webhook 增量同步
  - 不做 LRU 缓存清理（v1 留给后续）
  - 不加密 PAT（明文 SQLite）
- **限流硬阈值**：`X-RateLimit-Remaining ≤ 5` 时 short-circuit；阈值常量定义在 `GitHubClient` 中
- **大仓上限**：Trees API `truncated=true` 时直接抛 `TREE_TOO_LARGE`（≈100k 节点上限）

---

## 6. 下游依赖调用

### 外部 HTTP

```text
GET  https://api.github.com/repos/{o}/{r}/branches/{ref}          → branch info + commit.sha
GET  https://api.github.com/repos/{o}/{r}/git/trees/{sha}?recursive=1  → 整树
GET  https://raw.githubusercontent.com/{o}/{r}/{refSha}/{path}    → 文件正文
```

请求头：
- `User-Agent: kai-toolbox-doc-viewer/1.0`（GitHub API 必需）
- `Accept: application/vnd.github+json`
- `X-GitHub-Api-Version: 2022-11-28`
- `Authorization: token {pat}`（仅 pat 非空时附加）
- `If-None-Match: {etag}`（refresh 时附加）

### 内部 Bean 注入

```text
DocViewerController <- DocViewerService
DocViewerService <- GitHubClient + DocCacheRepository
DocCacheRepository <- JdbcTemplate（toolbox-common 已配置）
```

无新增 Maven 依赖（HttpClient 用 JDK 内建；Jackson 已在 toolbox-common）

---

## 7. 异常处理要点

| 场景 | 抛出 / 返回 |
|------|------------|
| URL 无法解析为 (owner, repo, ref) | `DocViewerException(INVALID_GITHUB_URL)` → 400 |
| GitHub 返 404 | `DocViewerException(REPO_NOT_FOUND)` → 404 |
| GitHub 返 401 / 403 + 私库 | `DocViewerException(REPO_FORBIDDEN)` → 403 |
| GitHub 返 403 + `X-RateLimit-Remaining=0` | 在 `getTreeIfModified` 转为 `RateLimited` 信号；Service 写 `rate_limit_until` 返缓存（不抛异常）；若初次 createSource 时无缓存则抛 `RATE_LIMITED` → 429 |
| Trees `truncated=true` | `DocViewerException(TREE_TOO_LARGE)` → 400 |
| sourceId 不存在 | `DocViewerException(SOURCE_NOT_FOUND)` → 404 |
| getFile 时 path 不在 tree_cache | `DocViewerException(FILE_NOT_IN_TREE)` → 404，前端 toast 提示"刷新后重试" |
| 网络/IOException | `DocViewerException(UPSTREAM_UNAVAILABLE)` → 502 |
| `DocViewerException` → JSON | 由 `toolbox-common.GlobalExceptionHandler` 统一处理；body `{code: errorCodeName, message: i18nKey}` |

---

## 8. 前端实现要点（补充）

- **mermaid 懒加载**：`MermaidRenderer` 内部 `await import('mermaid')`，首次见到 mermaid 代码块才下载 chunk
- **相对链接重写规则**（`rehype-rewrite`）：
  - `<img src="./x.png">` → `<img src="{rawBaseUrl}{currentDir}/x.png">`
  - `<a href="./x.md">` → `<a href="/tools/doc-viewer/{sourceId}/{currentDir}/x.md">`
  - `<a href="https://...">` 保持原样
  - `<a href="#anchor">` 保持原样（页内跳转）
- **目录优先索引文件**（FileTree 点目录时）：
  ```typescript
  const indexFile = ['INDEX.md', 'README.md', '00_index.md']
    .map(name => `${dirPath}/${name}`)
    .find(p => treeNodeMap.has(p));
  if (indexFile) navigate(indexFile);
  ```
- **大文件渐进策略**（MarkdownView mount 时）：
  ```typescript
  const initial = chooseInitialViewMode(size);  // > 2MB 走 raw
  if (showSkeletonHint(size)) {
    // 显骨架 → setTimeout(() => setReady(true), 0) 让骨架先 paint
  }
  ```
