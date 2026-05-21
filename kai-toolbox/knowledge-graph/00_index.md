# kai-toolbox 知识图谱（索引）

> 本目录沉淀 kai-toolbox 后端单服务的技术难点、数据流约束、可复用模式。
> 候选事实先入 `_candidates.md`；代码验证后整理到 `scenarios/`。
> 跨项目链路（如本工具调外部服务）走 `cross-project-locator`，不在此目录。

## 场景索引

| 场景 | 主题 | 一句话 | 关键词 | 文件 |
|------|------|--------|--------|------|
| ffmpeg 子进程编排与超时控制 | 进程生命周期 | stream drain 必须后台 + 主线程 waitFor + destroyForcibly + 启动期 reaper + JVM 退出 hook | ffmpeg / 缩略图 / HLS / probe / 子进程 / 孤儿 / timeout / transferTo / Semaphore | [scenarios/ffmpeg子进程编排与超时控制.md](scenarios/ffmpeg子进程编排与超时控制.md) |
| 异步任务编排与进度推送 | 任务管线 / 状态机 / SSE | 两类任务两套线程池 + AtomicBoolean cancel + ProcessRegistry 杀子进程 + per-job 单订阅 SSE + 全局多订阅 TaskBroadcaster + SQLite 状态镜像 | 字幕作业 / 目录扫描 / 虚拟线程 / 平台线程 / cancelFlags / SseEmitterRegistry / TaskBroadcaster / TaskAssembler / TaskView / 任务中心 / 状态机 / SSE 事件 | [scenarios/异步任务编排与进度推送.md](scenarios/异步任务编排与进度推送.md) |

## 待补的场景卡

按需要随时新增，目前已识别但未沉淀的主题：

- 视频识别与播放协议路由（probe → native vs HLS 的判定矩阵）
- 视频缩略图缓存键策略（SHA-1(path + mtime)、`.tmp` 原子改名、`.failed` marker、上限 LRU）
- 文件路径越权防护（toRealPath + startsWith，跨平台路径与 symlink）
- HLS 分片切割与首字节延迟（fast-path copy vs encode、preset 选择、客户端断开清理）
- 资源争夺与让位机制（CPU/disk/Semaphore/ActivePlaybackTracker 协作）

## 关键词反查

| 关键词 | 命中场景 |
|--------|---------|
| ffmpeg / ffprobe / 子进程 / 孤儿 / 0% CPU | ffmpeg 子进程编排与超时控制 |
| transferTo / waitFor / destroyForcibly / drain | ffmpeg 子进程编排与超时控制 |
| ProcessHandle / 启动期 reaper / shutdown hook | ffmpeg 子进程编排与超时控制 |
| Semaphore / maxParallel / fair queue | ffmpeg 子进程编排与超时控制 |
| `.tmp` / `.failed` marker | ffmpeg 子进程编排与超时控制 |
| 字幕作业 / SubtitleJob / TRANSCRIBING / TRANSLATING / ANALYZING_AUDIO | 异步任务编排与进度推送 |
| 目录扫描 / ScanRecord / ScanStatus / RUNNING | 异步任务编排与进度推送 |
| 平台线程 / 虚拟线程 / newFixedThreadPool / newThreadPerTaskExecutor | 异步任务编排与进度推送 |
| AtomicBoolean / cancelFlags / 协作式取消 / 启停管理 | 异步任务编排与进度推送 |
| SseEmitter / SseEmitterRegistry / TaskBroadcaster / 多订阅 / 单订阅 | 异步任务编排与进度推送 |
| TaskView / TaskAssembler / 任务中心 / `/tasks/events` | 异步任务编排与进度推送 |
| SSE 事件名 / status / progress / completed / language / analysis | 异步任务编排与进度推送 |
| spring.mvc.async.request-timeout / SSE 长连 | 异步任务编排与进度推送 |
| 删除串行化 / deletionLock / 4 表事务 | 异步任务编排与进度推送 |

## 候选沉淀池

会话中提到、但未代码验证或未整理为场景卡的事实，落在 [_candidates.md](_candidates.md)。
SQL / 查询逻辑候选（kai-toolbox 几乎不直接写业务 SQL，主要走 Spring JDBC + 简单 SELECT/INSERT，暂不开 `_sql_candidates.md`）。

## 与设计文档的边界

| 类型 | 路径 | 用途 |
|------|------|------|
| 设计文档（业务 / 技术方案） | `{USER_DOCUMENTS}/ai-docs/kai-toolbox/design/{需求}/` | 怎么设计、为什么这么设计 |
| **知识图谱（本目录）** | `{USER_DOCUMENTS}/ai-docs/kai-toolbox/knowledge-graph/` | **既有事实**：可复用模式、状态判定、子进程编排约束、关键代码坐标 |
| 项目 INDEX | `kai-toolbox/docs/design/` | 用户上传终版后才走，目前空 |
