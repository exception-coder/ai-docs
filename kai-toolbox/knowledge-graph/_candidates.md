# kai-toolbox 知识图谱候选沉淀池

> 会话中提到、未完全验证或还未整理为正式场景卡的事实，先在这里累积。
> 后续如果用户要求"整理图谱 / 归档"，从这里读取去重合并到 `scenarios/`。

---

## 2026-05-06 00:48 | ffmpeg 子进程"流排水 + 超时"陷阱

- 事实：Java `InputStream.transferTo()` 阻塞直到 EOF；ffmpeg stdout EOF 仅在进程退出时出现。主线程同步 drain 会让 `waitFor(timeout)` 变死代码，ffmpeg 卡死时进程坐着 0% CPU 占满 semaphore 槽。
- 涉及：`ThumbnailService.runFfmpeg`、`FfmpegProbe.runFfprobe`
- 来源：实战 bug，用户截图 2 个 ffmpeg 0% CPU、94MB / 16MB 内存仍在跑
- 可信度：已代码验证（修复后 stream drain 推到虚拟线程，主线程 waitFor 强制超时）
- 后续动作：**已整理入 `scenarios/ffmpeg子进程编排与超时控制.md`**

## 2026-05-06 01:00 | JVM 强杀产生孤儿 ffmpeg

- 事实：`ProcessBuilder.start()` 出来的子进程不是 JVM 逻辑子进程，JVM 退出后变孤儿继续跑（Windows 尤其明显）。`taskkill /F` / `kill -9` 不可拦截，`Runtime.addShutdownHook` 和 `@PreDestroy` 都不会触发。
- 缓解：`FfmpegProcessRegistry@PostConstruct` 启动期扫 `ProcessHandle.allProcesses()` 找出绝对路径匹配 `toolbox.ffmpeg.binary` 的进程并 destroyForcibly；同时 `Runtime.addShutdownHook` + `@PreDestroy` 双重保险捕获优雅退出。
- 涉及：`FfmpegProcessRegistry`
- 来源：用户截图 4 个孤儿 + 实战 bug
- 可信度：已代码验证
- 后续动作：**已整理入 `scenarios/ffmpeg子进程编排与超时控制.md`**

## 2026-05-06 01:15 | CPU 资源争夺导致 HLS 播放卡顿

- 事实：ffmpeg 进程是真并行（OS 进程级），但 4 个 HEVC 解码 ffmpeg 同时跑会榨干 CPU，HLS 播放的第 5 个 ffmpeg 拿不到时间片 → 分片生成慢于实时 → hls.js 缓冲耗尽 → 播放卡顿。**不是软件层互锁**，纯 OS 资源争夺。
- 缓解：
  - `ThumbnailProperties.maxParallel` 默认从 4 降到 2，留 CPU 给 HLS
  - `ActivePlaybackTracker.touch()` 在 controller 播放路径打时间戳；`ThumbnailWarmer` 每次迭代前检查 `recentlyActive(15s)`，活跃则 sleep 2s 重轮询，让位 CPU
- 涉及：`ThumbnailWarmer`、`ActivePlaybackTracker`、`TreeSizeController` 各播放端点
- 可信度：已代码验证
- 后续动作：**已整理入 `scenarios/ffmpeg子进程编排与超时控制.md`** 的"让位 / 让步"小节

## 2026-05-06 00:30 | 路径越权与系统目录过滤

- 事实：所有按 `path` 参数读盘的接口必须先过 `PathAccessGuard.resolve(scanId, path)`。它做：(1) 拿 scan rootPath `toRealPath()`（解 symlink）；(2) 请求 path `toRealPath()`；(3) 要求后者 `startsWith` 前者；(4) 必须是 `Files.isRegularFile`。
- 视频库聚合查询额外排除 OS 目录：`$RECYCLE.BIN`、`System Volume Information`、`RECYCLER`、`.Trashes`、`.Trash-`，避免 macOS metadata、Windows 回收站文件污染列表。`findChildren`（TreeSize 浏览）不过滤——用户想看回收站占用是合理诉求。
- 涉及：`PathAccessGuard`、`NodeRepository.findVideos / findJunkVideos` 的 SQL `NOT LIKE` 黑名单
- 可信度：已代码验证
- 后续动作：可整理入独立场景卡 `scenarios/路径越权与系统目录过滤.md`（待）

## 2026-05-17 14:00 | 异步任务编排与启停 / 进度推送 三件套

- 事实：后端两类异步任务（SubtitleJob 字幕作业、ScanRecord 目录扫描）跑在**按工作负载分裂**的两套线程池上——字幕用 `Executors.newFixedThreadPool` 平台线程（GPU-bound 硬上限），扫描用 `newThreadPerTaskExecutor + Thread.ofVirtual()`（IO-bound 弹性）。
- 启停：每任务一个 `AtomicBoolean cancelled`，注册到 `ConcurrentHashMap<id, AtomicBoolean>`；`cancel(id)` 只置 flag 不打断线程；worker 在安全点 poll 自检 → throw InterruptedException → finally 清理。whisper / ffmpeg 子进程**额外**通过 `FfmpegProcessRegistry` 主动 destroyForcibly。
- 通知前端：两条 SSE 频道并行存在——`SseEmitterRegistry`（per-jobId 单订阅，详情页用）+ `TaskBroadcaster`（全局多订阅 Set\<SseEmitter\>，任务中心用）。每次状态变化 worker 同时 publish 给两边。事件名约定见正式卡。
- 状态权威是 SQLite：每次 status / progress 切换都立即 `UPDATE`，刷新页面先 GET 快照再 subscribe SSE 接续。`application.yml: spring.mvc.async.request-timeout: -1` 保证长连。
- 涉及：`SubtitleService` / `ScanService` / `SseEmitterRegistry` / `TaskBroadcaster` / `TaskAssembler` / `TreeSizeController.listTasks/taskEvents`
- 可信度：已代码验证（任务中心模块开发本会话内完整改完 + mvn / typecheck / build 全绿）
- 后续动作：**已整理入 `scenarios/异步任务编排与进度推送.md`**
- 待补：JVM 强杀留下的"僵尸 active 任务"启动期 reaper（把 ANALYZING_AUDIO / TRANSCRIBING / TRANSLATING / RUNNING 状态一律转 FAILED），目前依赖用户手动重跑覆盖。高频进度回调未节流，单用户够用，多用户需补。

## 2026-05-06 00:30 | `.ts` 扩展名的视频库特殊处理

- 事实：`.ts` 既是 MPEG-TS 视频容器（HLS 分片格式），也是 TypeScript 源码扩展名。在 dev 机器上后者占绝大多数。
- 处理：保留在全局白名单 `toolbox.video.extensions`（让 TreeSize 点击 .ts 仍能走播放器），但**视频库聚合 endpoint** `libraryVideos` 内显式 `filter(e -> !"ts".equalsIgnoreCase(e))` 排除掉。`junk-clean` 不过滤（让 macOS `._foo.ts` 元数据也能清理）。
- 涉及：`TreeSizeController.libraryVideos`
- 可信度：已代码验证
- 后续动作：可整理入独立场景卡（待）
