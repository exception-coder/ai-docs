# Docker 服务治理 - 编码摘要

> 由 `Docker服务治理-current.md`（完整-技术）精简而来，聚焦实现所需的最小必要信息。
> 字段级契约见 `Docker服务治理-api-current.md`，本文档不重复。

---

## 1. 核心业务规则

1. 应用唯一键 `(host_id, base_dir)`，DB 加 `UNIQUE` 索引；扫描结果命中则标 `registered=true`。
2. action 白名单：容器 `start/stop/restart/pause/unpause/kill`，compose `up/down/restart/pull`；非白名单 400。
3. 路径白名单：所有 `path` 经 `readlink -f` 解析后必须以 `realpath(baseDir) + '/'` 为前缀。
4. 单文件读写上限 256KB；超出 413。
5. 所有用户输入（容器 id、路径、文件名）一律走 `HostSshExec.singleQuote`，禁字符串插值。
6. 除日志 follow 外，所有 SSH session 在方法返回前关闭（try-with-resources / finally）。
7. SSE `onCompletion/onTimeout/onError` 全部触发 `HostSshStream.close()`；空闲 30 分钟自动断流。
8. `DELETE /apps/{appId}` 不调任何远端命令。
9. 创建应用且 `skipValidate=false` 时跑 `docker compose -f {file} config -q`，失败 422 + stderr。
10. `docker compose` vs `docker-compose` 命令探测一次，结果按 hostId 缓存到内存 Map（启动后首次访问该 host 时探测）。
11. SSE 日志限速：单连接每秒最多 1000 行，超出丢弃头部。
12. 容器列表 / stats 后端 30s TTL 缓存：key = `containers:{hostId}:{appId?}:{includeStopped}` 与 `stats:{hostId}`；本工具写操作（容器 / compose 动作）执行成功后立刻 `invalidateHost(hostId)`；Query `?nocache=true` 跳过缓存读但读后仍写入缓存。

---

## 2. 接口入口指针

> 字段级契约见 `Docker服务治理-api-current.md`。

| 接口 | 实现类#方法 |
|------|-------------|
| `GET    /api/docker/hosts/{hostId}/apps` | `DockerAppController#list` |
| `POST   /api/docker/hosts/{hostId}/apps` | `DockerAppController#create` |
| `PUT    /api/docker/hosts/{hostId}/apps/{appId}` | `DockerAppController#update` |
| `DELETE /api/docker/hosts/{hostId}/apps/{appId}` | `DockerAppController#delete` |
| `POST   /api/docker/hosts/{hostId}/scan` | `DockerAppController#scan` |
| `GET    /api/docker/hosts/{hostId}/containers` | `DockerContainerController#list` |
| `POST   /api/docker/hosts/{hostId}/containers/{cid}/{action}` | `DockerContainerController#containerAction` |
| `GET    /api/docker/hosts/{hostId}/containers/stats` | `DockerContainerController#stats` |
| `POST   /api/docker/hosts/{hostId}/apps/{appId}/compose/{action}` | `DockerContainerController#composeAction` |
| `GET    /api/docker/hosts/{hostId}/containers/{cid}/logs` | `DockerLogController#tail` |
| `GET    /api/docker/hosts/{hostId}/containers/{cid}/logs/stream` | `DockerLogController#follow` |
| `DELETE /api/docker/streams/{streamId}` | `DockerLogController#closeStream` |
| `GET    /api/docker/hosts/{hostId}/apps/{appId}/files` | `DockerConfigController#listFiles` |
| `GET    /api/docker/hosts/{hostId}/apps/{appId}/files/content` | `DockerConfigController#readFile` |
| `PUT    /api/docker/hosts/{hostId}/apps/{appId}/files/content` | `DockerConfigController#writeFile` |

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.toolbox.docker.config.DockerToolDescriptor` | 新建 | `ToolDescriptor` 实现 |
| `com.exceptioncoder.toolbox.docker.api.DockerAppController` | 新建 | 应用 CRUD + 扫描 |
| `com.exceptioncoder.toolbox.docker.api.DockerContainerController` | 新建 | 容器/compose 控制 + stats |
| `com.exceptioncoder.toolbox.docker.api.DockerLogController` | 新建 | tail / SSE follow |
| `com.exceptioncoder.toolbox.docker.api.DockerConfigController` | 新建 | 配置文件读写 |
| `com.exceptioncoder.toolbox.docker.api.dto.DockerAppRequest` | 新建 | 创建/更新入参 |
| `com.exceptioncoder.toolbox.docker.api.dto.DockerAppView` | 新建 | 出参 |
| `com.exceptioncoder.toolbox.docker.api.dto.ScannedAppView` | 新建 | 扫描结果元素 |
| `com.exceptioncoder.toolbox.docker.api.dto.ContainerView` | 新建 | 容器出参 |
| `com.exceptioncoder.toolbox.docker.api.dto.ContainerStatsView` | 新建 | 资源快照单条 |
| `com.exceptioncoder.toolbox.docker.api.dto.ComposeFileView` | 新建 | 文件清单单条 |
| `com.exceptioncoder.toolbox.docker.api.dto.FileContentRequest` | 新建 | 写文件入参 |
| `com.exceptioncoder.toolbox.docker.api.dto.FileContentView` | 新建 | 读文件出参 |
| `com.exceptioncoder.toolbox.docker.api.dto.ContainerActionResponse` | 新建 | compose 动作出参（含 exitCode/stdout/stderr/durationMs） |
| `com.exceptioncoder.toolbox.docker.domain.DockerApp` | 新建 | Lombok `@Data @Builder` |
| `com.exceptioncoder.toolbox.docker.domain.ContainerAction` | 新建 | enum (`START/STOP/RESTART/PAUSE/UNPAUSE/KILL`) + `toCommand()` |
| `com.exceptioncoder.toolbox.docker.domain.ComposeAction` | 新建 | enum (`UP/DOWN/RESTART/PULL`) + `toCommand(ComposeOptions)` |
| `com.exceptioncoder.toolbox.docker.repository.DockerAppRepository` | 新建 | SQLite CRUD + 唯一性查询 |
| `com.exceptioncoder.toolbox.docker.service.DockerService` | 新建 | 编排所有远端命令 |
| `com.exceptioncoder.toolbox.docker.service.DockerComposeScanner` | 新建 | 扫描发现 compose |
| `com.exceptioncoder.toolbox.docker.service.DockerPsParser` | 新建 | 解析 docker ps JSON |
| `com.exceptioncoder.toolbox.docker.service.HostSshStream` | 新建 | 长连接 SSH 流封装 |
| `com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder` | 新建 | 命令拼装 + 转义 + cli 版本判定 |
| `com.exceptioncoder.toolbox.docker.service.ContainerCache` | 新建 | 容器列表 / stats 的 30s TTL 缓存 + host 维度失效 |

### 关键方法签名

```
// Service
com.exceptioncoder.toolbox.docker.service.DockerService#listApps(String hostId): List<DockerApp>
com.exceptioncoder.toolbox.docker.service.DockerService#createApp(String hostId, DockerAppRequest req): DockerApp
com.exceptioncoder.toolbox.docker.service.DockerService#updateApp(String hostId, String appId, DockerAppRequest req): DockerApp
com.exceptioncoder.toolbox.docker.service.DockerService#deleteApp(String hostId, String appId): void
com.exceptioncoder.toolbox.docker.service.DockerService#scan(String hostId, String baseDir, int maxDepth): List<ScannedAppView>
com.exceptioncoder.toolbox.docker.service.DockerService#listContainers(String hostId, String appIdOrNull, boolean includeStopped, boolean noCache): List<ContainerView>
com.exceptioncoder.toolbox.docker.service.DockerService#containerAction(String hostId, String cid, ContainerAction action): void  // 成功后 cache.invalidateHost(hostId)
com.exceptioncoder.toolbox.docker.service.DockerService#stats(String hostId, boolean noCache): List<ContainerStatsView>
com.exceptioncoder.toolbox.docker.service.DockerService#composeAction(String hostId, String appId, ComposeAction action, ComposeOptions opts): ContainerActionResponse  // 成功后 cache.invalidateHost(hostId)
com.exceptioncoder.toolbox.docker.service.DockerService#tailLogs(String hostId, String cid, int tail, String since, boolean timestamps): List<String>
com.exceptioncoder.toolbox.docker.service.DockerService#followLogs(String hostId, String cid, int tail, String since, boolean timestamps, SseEmitter emitter): String /* streamId */
com.exceptioncoder.toolbox.docker.service.DockerService#closeStream(String streamId): void
com.exceptioncoder.toolbox.docker.service.DockerService#listFiles(String hostId, String appId): List<ComposeFileView>
com.exceptioncoder.toolbox.docker.service.DockerService#readFile(String hostId, String appId, String path): FileContentView
com.exceptioncoder.toolbox.docker.service.DockerService#writeFile(String hostId, String appId, String path, String content): FileContentView

// Scanner / Parser
com.exceptioncoder.toolbox.docker.service.DockerComposeScanner#scan(Session session, String baseDir, int maxDepth): List<ScannedAppView>
com.exceptioncoder.toolbox.docker.service.DockerPsParser#parse(String jsonLines, Map<String,String> appIdByProject): List<ContainerView>
com.exceptioncoder.toolbox.docker.service.DockerPsParser#parseStats(String jsonLines): List<ContainerStatsView>

// Stream
com.exceptioncoder.toolbox.docker.service.HostSshStream#open(Session session, String command, Consumer<String> onLine, Runnable onComplete): HostSshStream
com.exceptioncoder.toolbox.docker.service.HostSshStream#close(): void

// ContainerCache
com.exceptioncoder.toolbox.docker.service.ContainerCache#get(String key, Supplier<T> loader, Class<T> type): T  // 30s TTL
com.exceptioncoder.toolbox.docker.service.ContainerCache#invalidateHost(String hostId): void
com.exceptioncoder.toolbox.docker.service.ContainerCache#put(String key, Object value): void  // nocache 读后回填

// CommandBuilder
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#composeBin(String hostId, Session session): String  // 探测 + 缓存
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#dockerPs(boolean includeStopped): String
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#dockerLogs(String cid, int tail, String since, boolean ts, boolean follow): String
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#dockerStats(): String
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#compose(String baseDir, String composeFile, String composeBin, ComposeAction a, ComposeOptions o): String
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#find(String baseDir, int maxDepth): String
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#realpath(String path): String
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#listConfigFiles(String baseDir): String
com.exceptioncoder.toolbox.docker.service.DockerCommandBuilder#backupAndWrite(String path, String tmpPath, String backupPath): String[]  /* 返回三条命令：cp 备份 / tee 临时 / mv 原子替换 */

// Repository
com.exceptioncoder.toolbox.docker.repository.DockerAppRepository#findAllByHost(String hostId): List<DockerApp>
com.exceptioncoder.toolbox.docker.repository.DockerAppRepository#findById(String id): Optional<DockerApp>
com.exceptioncoder.toolbox.docker.repository.DockerAppRepository#existsByHostAndBaseDir(String hostId, String baseDir): boolean
com.exceptioncoder.toolbox.docker.repository.DockerAppRepository#insert(DockerApp app): void
com.exceptioncoder.toolbox.docker.repository.DockerAppRepository#update(DockerApp app): void
com.exceptioncoder.toolbox.docker.repository.DockerAppRepository#deleteById(String id): void
```

---

## 4. 数据结构

### 4.1 SQLite 新增表

```sql
-- tools/tool-docker/src/main/resources/db/docker-schema.sql
CREATE TABLE IF NOT EXISTS docker_app (
    id           TEXT PRIMARY KEY,
    host_id      TEXT NOT NULL,
    name         TEXT NOT NULL,
    base_dir     TEXT NOT NULL,
    compose_file TEXT NOT NULL DEFAULT 'docker-compose.yml',
    note         TEXT,
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_docker_app_host ON docker_app(host_id);
CREATE UNIQUE INDEX IF NOT EXISTS uk_docker_app_host_dir ON docker_app(host_id, base_dir);
```

### 4.2 关键 DTO

```java
// DockerApp（domain）
String id;             // UUID
String hostId;
String name;
String baseDir;
String composeFile;    // 默认 docker-compose.yml
String note;
long createdAt;
long updatedAt;

// ContainerView（dto）
String id;
String shortId;        // id 前 12 位
String name;
String image;
String state;          // running / exited / paused / created / restarting
String status;
long createdAt;        // unix 秒
String ports;
String composeProject; // 可空
String composeService; // 可空
String appId;          // 反查命中的应用，可空

// ComposeOptions（service 内部）
boolean detach = true;
boolean removeOrphans = false;
String pullPolicy = "missing";
```

### 4.3 内存缓存

```java
// DockerCommandBuilder 内部
private final ConcurrentMap<String, String> composeBinCache = new ConcurrentHashMap<>();
// key = hostId, value = "docker compose" or "docker-compose"
```

---

## 5. 重要约束与边界

- **会话生命周期**：除 `followLogs` 外，每个 public service 方法体 = `try (var s = sessions.open(host)) { ... }`。`JSch Session` 无 AutoCloseable，需要自封一个 `try/finally` wrapper 或局部变量手动关。
- **SSE 注册流程**：`DockerLogController#follow` → `sseRegistry.create(streamId)` → `service.followLogs(..., emitter, streamId)` → service 内 `HostSshStream.open` 时把流挂到 `streamId` 映射，回调中调 `emitter.send`，并在 `emitter.onCompletion/onTimeout/onError` 触发 `HostSshStream.close` + 从映射移除。
- **路径校验**：`realpath` 远端命令实现 = `readlink -f -- {q(path)}`；失败（如 path 不存在）按 404 处理；存在但前缀不匹配按 403。
- **SSE 行编码**：`emitter.send(SseEmitter.event().name("log").data(Base64.encode(line)))`；客户端 `atob(e.data)` 还原。
- **限速**：`HostSshStream` 内部维护 `lastSecondLines` 计数器，跨秒清零；溢出行不入队（视为丢弃）。
- **`SchemaInitializer` 约定**：所有 DDL 必须 `CREATE TABLE IF NOT EXISTS` / `CREATE INDEX IF NOT EXISTS`，禁止 `ALTER TABLE`（无幂等保证）。后续字段变更需要走「新建表 + 数据迁移 + drop 老表」三段式或额外 Migration 机制（暂未引入）。
- **跨模块依赖**：`tool-docker/pom.xml` 显式依赖 `tool-hosts`；不直接访问 `host` 表，而是通过 `HostsService.findRequired(id)`。
- **`pom.xml` 与 `toolbox-starter/pom.xml`**：根 pom 的 `<modules>` 加 `tools/tool-docker`；starter 的 `<dependency>` 也加；遗漏任一会导致工具不被启动器加载。
- **前端 Manifest**：`frontend/src/features/docker/index.tsx` 用 lucide 的 `Boxes` 组件引用，**禁止用字符串**——`featureRegistry` 不做字符串→组件映射。
- **TanStack Query keys**：约定为 `['docker', 'apps', hostId]` / `['docker', 'containers', hostId, appId]` / `['docker', 'stats', hostId]` / `['docker', 'files', hostId, appId]`；动作完成后按这些前缀 `invalidateQueries`。容器/stats 查询 `staleTime: 30_000`，与后端缓存联动。「刷新」按钮 → `refetch({ throwOnError: true })` 并在 URL 加 `nocache=true`。
- **ContainerCache 实现**：`record Entry(Object value, long expireAtMs)`，`ConcurrentHashMap<String, Entry>` + `String.intern()` 上的 synchronized 块串行化同 key 回源；TTL 常量 30_000ms；`invalidateHost(hostId)` 用 `entrySet().removeIf(k -> k.startsWith("containers:"+hostId+":") || k.equals("stats:"+hostId))`。不引入 Caffeine（避免 premature infrastructure，CLAUDE.md §约定）。
- **CodeMirror 集成**：复用项目已有 `@uiw/react-codemirror` + `@codemirror/lang-yaml/json`，不引 Monaco。language extension 推断函数集中放在 `frontend/src/features/docker/lib/composeLang.ts`：`*.yml/.yaml/compose.y*ml/docker-compose.y*ml` → `yaml()`；`*.json` → `json()`；其余返回空数组（纯文本）。
- **SseEmitterRegistry chain 模式**：`SseEmitterRegistry.create(key)` 内部已 set 了 `onCompletion → emitters.remove(key, emitter)`。Controller 拿到 emitter 后**再次 set** `onCompletion` 会**覆盖** registry 的回调；正确做法是把两件事合写：`emitter.onCompletion(() -> { registry.complete(streamId); logStreamRegistry.close(streamId); })`，`onTimeout`/`onError` 同理。`registry.complete()` 内已对二次 complete 做 try/catch。

---

## 6. 单测与验证要点

- **路径白名单**：`baseDir=/opt/dockerApps/nginx` 时，`path=/opt/dockerApps/nginx/../redis/config` 必须 403；`path=/opt/dockerApps/nginx/sub/file.yml` 必须放行；软链指向 baseDir 外必须 403。
- **action 注入**：`POST .../containers/$(rm)/restart` 中的特殊字符必须由 Spring path 解析层过滤或在 Service 校验 `cid.matches("[a-zA-Z0-9_.-]{1,128}")`。
- **compose vs docker-compose 探测**：第一次调用打两条命令；之后命中缓存只打一条。
- **SSE 断流**：浏览器关闭页面 5 秒内服务端日志可见 `STREAM closed` 行；30 分钟空闲也能自然关闭。
- **唯一键冲突**：连续两次创建相同 `(hostId, baseDir)` 第二次必须 409。
- **256KB 上限**：上传 257KB 内容必须 413，且不动远端文件。
