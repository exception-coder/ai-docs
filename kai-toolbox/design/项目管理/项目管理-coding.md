# 项目管理 - 编码摘要

> 最后更新：2026-05-11
> 配套文档：`项目管理-current.md`、`项目管理-api-current.md`

---

## 1. 核心业务规则（提取自 current §3）

| 编号 | 规则 |
|---|---|
| R1 | 仅扫描一级子目录，不递归 |
| R2 | 跳过以 `.` `_` 开头的目录（可配置 `toolbox.projects.hidden-prefixes`） |
| R3 | 类型识别按优先级首命中：flutter > maven > gradle > node > python > git > other |
| R4 | `.git/HEAD` 读取失败必须静默，branch 字段返回 null |
| R5 | 后端缓存 TTL 5 秒（`toolbox.projects.cache-ttl-seconds`） |
| R6 | autorun 仅允许白名单值 `claude` |
| R7 | autorun 在前端发送，不进入 Web 终端后端契约 |
| R8 | 列表按 `lastModified` 倒序 |
| R9 | `/api/projects/open` 必须校验 `Path.startsWith(root)` |
| R10 | 根目录不存在时返回空列表 + `rootExists=false`，不抛 500 |

---

## 2. 接口入口指针

| 方法 | 路径 | 实现类#方法 |
|---|---|---|
| GET | `/api/projects` | `com.exceptioncoder.toolbox.projects.api.ProjectsController#list()` |
| POST | `/api/projects/open` | `com.exceptioncoder.toolbox.projects.api.ProjectsController#openInExplorer(OpenInExplorerRequest)` |

> 字段级契约见 `项目管理-api-current.md`。

---

## 3. 涉及类清单

### 3.1 后端（`tools/tool-projects`，包根 `com.exceptioncoder.toolbox.projects`）

#### 3.1.1 `config.ProjectsDescriptor`

```java
package com.exceptioncoder.toolbox.projects.config;

@Component
public class ProjectsDescriptor implements ToolDescriptor {
    @Override public String id() { return "projects"; }
    @Override public String name() { return "项目管理"; }
    @Override public String icon() { return "folder-git-2"; }
    @Override public int order() { return 10; }
}
```

参考既有 `WebTermDescriptor` 的写法保持一致。

#### 3.1.2 `config.ProjectsProperties`

跟 `WebTermProperties` 同款风格（class + Lombok），主类已有 `@ConfigurationPropertiesScan` + 自身 `@Component` 双重保险，无需额外 `@EnableConfigurationProperties`。

```java
@Component
@ConfigurationProperties(prefix = "toolbox.projects")
@Getter
@Setter
public class ProjectsProperties {
    private String root;
    private int cacheTtlSeconds = 5;
    private List<String> hiddenPrefixes = List.of(".", "_");
}
```

#### 3.1.3 `api.ProjectsController`

| 方法 | 签名 | 关键操作 |
|---|---|---|
| `list` | `GET /api/projects → ProjectsListResponse` | 调 `ProjectsCache.getOrLoad(scanner::scan)`；包装顶层字段 |
| `openInExplorer` | `POST /api/projects/open(OpenInExplorerRequest) → ResponseEntity<Void>` | 1) `Path.of(req.path).normalize()`；2) 校验 `startsWith(root)` 否则 400 `INVALID_PATH`；3) `Files.isDirectory` 否则 400；4) `Desktop.getDesktop().open(file)`；5) 捕获 `HeadlessException` → 503 |

错误码 → HTTP 映射通过抛业务异常 + `GlobalExceptionHandler` 转译，不直接 `ResponseEntity` 拼。

#### 3.1.4 `api.dto.ProjectInfo`（record）

```java
public record ProjectInfo(
    String name,
    String path,
    ProjectType type,
    String branch,        // nullable
    OffsetDateTime lastModified
) {}
```

#### 3.1.5 `api.dto.ProjectsListResponse`（record）

```java
public record ProjectsListResponse(
    String root,
    boolean rootExists,
    OffsetDateTime scannedAt,
    List<ProjectInfo> items
) {}
```

#### 3.1.6 `api.dto.OpenInExplorerRequest`（record）

```java
public record OpenInExplorerRequest(@NotBlank String path) {}
```

#### 3.1.7 `api.dto.ProjectType`（enum）

```java
public enum ProjectType { flutter, maven, gradle, node, python, git, other }
```

JSON 序列化默认按枚举名（小写），与 API 文档一致。

#### 3.1.8 `service.ProjectScanner`

| 方法 | 签名 | 操作 |
|---|---|---|
| `scan` | `ProjectsListResponse scan()` | 1) 取 `root = Path.of(props.root)`；2) `rootExists = Files.isDirectory(root)`；若否，返回空 items；3) `Files.list(root)` 取一级子目录；4) 过滤隐藏前缀；5) 对每个目录：`type = ProjectTypeDetector.detect(dir)`；若 `Files.exists(dir.resolve(".git"))` 则 `branch = readGitBranch(dir)`；6) 收集 `ProjectInfo`；7) 按 `lastModified` 倒序；8) 组装 `ProjectsListResponse` 设 `scannedAt = OffsetDateTime.now()` |
| `readGitBranch` | `String readGitBranch(Path dir)` | 读 `dir/.git/HEAD`：若以 `ref: refs/heads/` 开头取分支名；否则返回 commit 短哈希；任何 IOException catch 返回 null |

#### 3.1.9 `service.ProjectTypeDetector`

```java
public final class ProjectTypeDetector {
    private static final List<Map.Entry<ProjectType, List<String>>> RULES = List.of(
        Map.entry(ProjectType.flutter, List.of("pubspec.yaml")),
        Map.entry(ProjectType.maven,   List.of("pom.xml")),
        Map.entry(ProjectType.gradle,  List.of("build.gradle", "build.gradle.kts", "settings.gradle")),
        Map.entry(ProjectType.node,    List.of("package.json")),
        Map.entry(ProjectType.python,  List.of("pyproject.toml", "requirements.txt"))
    );

    public static ProjectType detect(Path dir) {
        for (var rule : RULES)
            for (String f : rule.getValue())
                if (Files.exists(dir.resolve(f))) return rule.getKey();
        if (Files.isDirectory(dir.resolve(".git"))) return ProjectType.git;
        return ProjectType.other;
    }
}
```

#### 3.1.10 `service.ProjectsCache`

单条目 TTL 缓存。**不引入 Caffeine，避免新依赖**。

```java
@Component
public class ProjectsCache {
    private final ProjectsProperties props;
    private volatile ProjectsListResponse cached;
    private volatile Instant expireAt = Instant.EPOCH;
    private final Object lock = new Object();

    public ProjectsListResponse getOrLoad(Supplier<ProjectsListResponse> loader) {
        if (Instant.now().isBefore(expireAt) && cached != null) return cached;
        synchronized (lock) {
            if (Instant.now().isBefore(expireAt) && cached != null) return cached;
            ProjectsListResponse r = loader.get();
            cached = r;
            expireAt = Instant.now().plusSeconds(props.cacheTtlSeconds());
            return r;
        }
    }
}
```

### 3.2 前端（`frontend/src/features/projects/`）

| 文件 | 关键实现 |
|---|---|
| `index.tsx` | `FeatureManifest`：id=projects, name=项目管理, icon=FolderGit2, group=系统工具, order=5（沿用现有 group；order=5 让本工具排在 treesize order=10 之前作为日常入口） |
| `pages/ProjectsPage.tsx` | `useProjects()` 取数据；顶部搜索框（受控 input，对 `name` 做 `includes`）；空态：rootExists=false 显示「根目录不存在：{root}」；列表：`ProjectCard[]` 用 CSS Grid 自适应（`auto-fill, minmax(280px,1fr)`） |
| `components/ProjectCard.tsx` | 卡片整块点击 → `useNavigate()(`/tools/webterm?cwd=${encodeURIComponent(p.path)}&autorun=claude`)`；`stopPropagation` 隔离三个次要按钮：跳转 / 复制路径（`navigator.clipboard.writeText`）/ 在文件管理器打开（POST `/api/projects/open`） |
| `components/ProjectTypeBadge.tsx` | type → {label, colorClass, IconComponent} 映射；用 lucide 图标（`GitBranch`/`Coffee`/`Boxes`/`Hexagon`/`Snake`/...）即可 |
| `hooks/useProjects.ts` | `useQuery({queryKey:['projects'], queryFn:projectsApi.list, staleTime:5_000, refetchOnWindowFocus:true})` |
| `api.ts` | 用 `@/lib/api` 的 `http<T>` 封装：`listProjects()` / `openInExplorer(path)`；项目里没有 axios，统一走 fetch 包装 |
| `types.ts` | 与后端 DTO 对齐，type 用字面量联合 |

### 3.3 Web 终端前端轻量改造（`frontend/src/features/webterm/`）

| 文件 | 改动 |
|---|---|
| `pages/WebTermPage.tsx` | 顶部 `const AUTORUN_WHITELIST = new Set(['claude'])`；`useSearchParams` 读 `cwd` `autorun`；`cwd` state 初始 = query.cwd；`autorun` state 初始 = query 在白名单内时取值，否则 null；用户手动改 `cwd` 输入框且 `autorun` 仍存在时 `setAutorun(null)`；点击「重新连接」也 `setAutorun(null)`；把 `autorun` 透传给 `Terminal` |
| `components/Terminal.tsx` | 新增 `autorun?: string \| null` prop；`useEffect(() => { if (state==='ready' && autorun) setTimeout(() => send(autorun+'\r\n'), 80) }, [state])`；本组件每个生命周期只 ready 一次，无需额外 ref 防重 |
| `hooks/useWebTermSocket.ts` | **不修改**。已暴露 `onReady` 回调与 `send` 接口；本次注入逻辑放在 Terminal 的 useEffect 内更内聚 |

> **不动**：`WebTermSocketHandler` / `WebTermSession` / `ShellLauncher` 等后端类。

---

## 4. 数据结构

无数据库变更。配置项：

```yaml
toolbox:
  projects:
    root: D:\Users\zhang\IdeaProjects
    cache-ttl-seconds: 5
    hidden-prefixes: [".", "_"]
```

---

## 5. 重要约束与边界

- **路径越权防护**：`/api/projects/open` 必须 `path.normalize().startsWith(root.normalize())`，且 `root.toAbsolutePath()`，避免 `..` 绕过
- **autorun 字面量校验**：前端 `WebTermPage` 严格 `if (autorun === 'claude')` 比较；任何其它值均忽略，不做泛化
- **git HEAD 读取超时**：`Files.readString` 默认无超时；本地文件 IO 不会卡，第一版不加超时
- **HeadlessException 处理**：仅在 `Desktop.isDesktopSupported() && Desktop.getDesktop().isSupported(OPEN)` 通过后才调用；否则直接 503
- **错误码统一**：所有业务错误抛 `BusinessException(code, msg)`（如已有），由 `GlobalExceptionHandler` 包装；本模块不裸抛 `RuntimeException`
- **缓存与 mtime 一致性**：5 秒缓存内某项目的 mtime 变化不会立刻反映；可接受
- **新增 Maven 模块需同步**：父 `pom.xml` `<modules>` + `<dependencyManagement>`；`toolbox-starter/pom.xml` `<dependencies>`；漏一处编译期就报错，不算运行时风险

---

## 6. 验证要点

| 场景 | 预期 |
|---|---|
| 进入 `/tools/projects`，根目录正常 | 卡片列表渲染，按 mtime 倒序，类型徽标正确 |
| 根目录不存在（改 `application.yml` 指向不存在路径） | 空态提示，HTTP 200 + `rootExists=false` |
| 同名重复请求（5 秒内多次刷新） | 后端日志只看到一次扫描日志 |
| 点击 `kai-toolbox` 卡片 | 跳转 Web 终端，cwd 为该项目，提示符显示 `PS D:\..\kai-toolbox>` 后自动出现 `claude` 命令并启动 |
| 在文件管理器打开 | 调起 Explorer 定位到目录 |
| 路径越权（手工 POST `/api/projects/open` 带 `C:\Windows`） | 400 `INVALID_PATH` |
| 复制路径按钮 | 剪贴板内容 = 项目绝对路径 |
| 隐藏前缀目录（`.git`、`_archive`） | 不出现在列表中 |
| 多签名共存（`flutter` + `git`） | type=flutter，branch 仍读取出来 |
| Web 终端无 query 直接进入 | 维持原默认行为（user.home + 不 autorun） |
