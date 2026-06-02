# 视频智能变速 编码摘要

> 由 `视频智能变速-current.md` 精简而来，聚焦「每个方法怎么写」。字段级接口契约见 `视频智能变速-api-current.md`。

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| current | 2026-06-02 | 初始版本：v1 整帧活动度 + 速度曲线 + 丢原音渲染 |

---

## 1. 核心业务规则

- 两阶段解耦：analyze 产曲线停在 `ANALYZED`；render 吃前端回传曲线直接渲，不重算分析。
- 状态机：`PENDING→ANALYZING→ANALYZED→RENDERING→DONE`，任意阶段可 `→FAILED/CANCELLED`。
- score→speed 分档：≥0.7→1.0x；≥0.4→1.5x；≥0.2→3.0x；<0.2→6.0x；FREEZE 段→8x（阈值/倍速进 Properties）。
- 最小段时长 `minSegmentSeconds`（默认 0.8s）合并防抖；ramp 仅在相邻速度差≥阈值的边界生成、限制子段数。
- **gap 语义**：render 的 segments 未覆盖区间直接跳过（不进输出）；区间不得重叠（重叠/speed≤0 → 400）。
- **产物临时**：DONE 产物落 `workDir/{jobId}/out.mp4`，过期由 `cleanStaleWorkDirs` 清理，不持久化、表不加产物列；重看需重渲。
- 音频：渲染恒定 `a=0` 丢原音；`musicPath` 非空才叠配乐，`-shortest` 对齐。
- 路径安全：`inputPath` 与 `musicPath` 均 `normalize()`+`isRegularFile`；产物下载 `normalize().startsWith(jobDir)`。
- 进程卫生：所有 ffmpeg 走 `FfmpegProcessRegistry.spawn` + 双虚拟线程 drain + `waitFor(timeout)` + `destroyForcibly` + 硬超时；取消即强杀。
- `progress`（0~1）为**内存瞬时值**（不落库），仅 ANALYZING/RENDERING 有意义。
- DDL 全 `CREATE TABLE/INDEX IF NOT EXISTS`。

---

## 2. 接口入口指针

| 接口 | 实现类 #方法 |
|------|-------------|
| `POST /api/video-condense/analyze` | `…api.VideoCondenseController#analyze` |
| `GET /api/video-condense/jobs/{id}` | `…api.VideoCondenseController#getJob` |
| `GET /api/video-condense/jobs/{id}/events` | `…api.VideoCondenseController#events` |
| `POST /api/video-condense/render` | `…api.VideoCondenseController#render` |
| `POST /api/video-condense/jobs/{id}/cancel` | `…api.VideoCondenseController#cancel` |
| `GET /api/video-condense/jobs/{id}/artifact` | `…api.VideoCondenseController#artifact` |
| `GET /api/video-condense/jobs` | `…api.VideoCondenseController#recent` |

> 包前缀 `com.exceptioncoder.toolbox.videocondense`。

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `…videocondense.api.VideoCondenseController` | 新建 | REST + SSE 适配，参数/路径校验 |
| `…videocondense.api.dto.AnalyzeRequest` | 新建 | `{ String path }` |
| `…videocondense.api.dto.RenderRequest` | 新建 | `{ String jobId; List<SegmentView> segments; String musicPath }` |
| `…videocondense.api.dto.JobView` | 新建 | 作业视图（含 progress、segments） |
| `…videocondense.api.dto.SegmentView` | 新建 | `{ double start,end,speed; String type; double score }` |
| `…videocondense.domain.CondenseJob` | 新建 | 作业实体（可变 status/error/curve） |
| `…videocondense.domain.JobStatus` | 新建 | 枚举 |
| `…videocondense.domain.SegmentType` | 新建 | 枚举 |
| `…videocondense.domain.ActivitySample` | 新建 | `record(double time, double score)` |
| `…videocondense.domain.Segment` | 新建 | `record(double start,end,score; SegmentType type; double speed)` |
| `…videocondense.domain.RenderSegment` | 新建 | `record(double start,end,speed)`（渲染只认这三个） |
| `…videocondense.service.CondenseJobService` | 新建 | 异步编排 + 状态机 + 内存运行态 + SSE |
| `…videocondense.service.ActivityAnalyzer` | 新建 | fork ffmpeg 抽帧算 scene 分 + freezedetect |
| `…videocondense.service.SegmentScorer` | 新建 | 活动度→分段+分类（纯函数） |
| `…videocondense.service.SpeedCurveGenerator` | 新建 | 分数→速度 + ramp（纯函数） |
| `…videocondense.service.FfmpegRenderService` | 新建 | setpts 分段变速 concat |
| `…videocondense.repository.CondenseJobRepository` | 新建 | Spring JDBC |
| `…videocondense.config.VideoCondenseToolDescriptor` | 新建 | 后端工具注册 |
| `…videocondense.config.VideoCondenseProperties` | 新建 | `toolbox.video-condense.*` |
| `pom.xml` / `toolbox-starter/pom.xml` | 修改 | 注册模块 + 依赖 |

### 关键方法签名与职责

```
// Controller —— 仅适配，业务全在 service
api.VideoCondenseController#analyze(AnalyzeRequest req): Map<String,String>          — 校验 path，返回 {jobId}
api.VideoCondenseController#getJob(String id): JobView                               — 查作业视图，404 不存在
api.VideoCondenseController#events(String id): SseEmitter                            — 注册 SSE，立即回当前快照
api.VideoCondenseController#render(RenderRequest req): JobView                       — 校验后置 RENDERING
api.VideoCondenseController#cancel(String id): JobView                               — 取消，409 已终态
api.VideoCondenseController#artifact(String id, HttpHeaders h): ResponseEntity<Resource> — Range 回放 out.mp4
api.VideoCondenseController#recent(int limit): List<JobView>                          — 最近作业

// 编排核心
service.CondenseJobService#analyze(String path): String                              — 建 PENDING 落库，虚拟线程跑 runAnalyze，返 jobId
service.CondenseJobService#runAnalyze(CondenseJob job): void                         — ANALYZING→probe→analyzer→scorer→generator→存曲线→ANALYZED；异常 FAILED
service.CondenseJobService#render(String jobId, List<RenderSegment> curve, String musicPath): void — 校验 ANALYZED+曲线合法，存曲线，RENDERING，虚拟线程跑 runRender
service.CondenseJobService#runRender(CondenseJob job, List<RenderSegment> curve, Path music): void — 调 render service→DONE(产物)/FAILED
service.CondenseJobService#cancel(String jobId): void                                — 标 CANCELLED + 强杀该 job 在跑的 Process
service.CondenseJobService#getJob(String id): JobView                                — 合并 DB 状态 + 内存 progress
service.CondenseJobService#publish(CondenseJob job): void                            — 组 JobView 经 SseEmitterRegistry.publish；终态 complete
service.CondenseJobService#validateCurve(List<RenderSegment>, double duration): void — speed>0、无重叠，否则 IllegalArgumentException

// 分析（fork ffmpeg）
service.ActivityAnalyzer#analyze(Path file, ProbeResult info, DoubleConsumer onProgress): AnalyzeResult
    — 一次 ffmpeg：fps=N,scale 低分辨率 + scene 分(metadata print) + freezedetect；解析 stderr/metadata
service.ActivityAnalyzer#parseSceneScores(...): List<ActivitySample>                 — 解析 lavfi.scene_score / pts_time
service.ActivityAnalyzer#parseFreezes(...): List<double[]>                           — freezedetect 起止区间

// 纯计算
service.SegmentScorer#score(List<ActivitySample> samples, List<double[]> freezes, double duration, VideoCondenseProperties p): List<Segment>
    — 按 windowSeconds 归并→打档→合并相邻同档→套 minSegmentSeconds→freeze 命中标 FREEZE
service.SpeedCurveGenerator#generate(List<Segment> segs, VideoCondenseProperties p): List<RenderSegment>
    — score→speed 分档 + 边界 ramp 线性插值子段
service.SpeedCurveGenerator#speedForScore(double score, SegmentType type, VideoCondenseProperties p): double

// 渲染（fork ffmpeg）
service.FfmpegRenderService#render(Path input, List<RenderSegment> curve, Path music, Path outDir): RenderOutcome
    — 生成 filter_complex_script→spawn ffmpeg→waitFor/destroyForcibly→产物校验
service.FfmpegRenderService#buildFilterScript(List<RenderSegment> curve): String     — 每段 trim+setpts，concat n=N v=1 a=0

// 持久化
repository.CondenseJobRepository#insert(CondenseJob job): void
repository.CondenseJobRepository#updateStatus(String id, JobStatus s, String error, long now): void
repository.CondenseJobRepository#updateCurve(String id, String curveJson, double durationSec, long now): void
repository.CondenseJobRepository#findById(String id): Optional<CondenseJob>
repository.CondenseJobRepository#findRecent(int limit): List<CondenseJob>
```

---

## 4. 数据结构

### 表 video_condense_job（`resources/db/video-condense-schema.sql`）

```sql
CREATE TABLE IF NOT EXISTS video_condense_job (
  id           TEXT PRIMARY KEY,
  input_path   TEXT NOT NULL,
  status       TEXT NOT NULL,              -- JobStatus.name()
  duration_sec REAL,                       -- 探测时长，未知为 NULL
  curve_json   TEXT,                       -- List<SegmentView> 序列化；ANALYZED 后有值
  error        TEXT,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_vcj_status ON video_condense_job(status);
CREATE INDEX IF NOT EXISTS idx_vcj_created ON video_condense_job(created_at);
```

> 不存产物路径（产物临时）；progress 不落库（内存）。

### 关键 DTO/DO 字段

```java
// JobView（出参）
String jobId; String status; String inputPath;
Double durationSec; double progress;   // progress 来自内存运行态
List<SegmentView> segments; String error;

// SegmentView：start,end,speed,score(double) + type(String)
// RenderSegment（render 入参 + 渲染内部）：start,end,speed —— render 只读这三个
```

### Properties（`toolbox.video-condense.*`）

```
workDir                 工作目录（产物/中间件）
retainMinutes           产物保留分钟数（cleanStaleWorkDirs 用，默认 60）
analyzeTimeoutSeconds   分析硬超时（默认 600）
renderTimeoutSeconds    渲染硬超时（默认 1800）
sampleFps               分析抽帧 fps（默认 4）
sampleScale             分析降采样（默认 160:90）
windowSeconds           评分窗口（默认 1.0）
minSegmentSeconds       最小段时长（默认 0.8）
rampSeconds             ramp 过渡时长（默认 0.4）
maxSegments             段数上限，超则触发 filter_complex_script（默认 200）
speedTiers              score 阈值→speed 分档（默认 0.7/0.4/0.2 → 1/1.5/3/6，freeze→8）
```

---

## 5. 重要约束与边界

- 并发：复用 `FfmpegProcessRegistry` 的 Semaphore 限并发；analyze/render 各自虚拟线程，不阻塞 WS/HTTP。
- 取消：`CondenseJobService` 内存维护 `jobId → Process`（AtomicReference），cancel 时 `descendants().forEach(destroyForcibly) + destroyForcibly`。
- 事务：单表单行写，无跨表事务；状态流转用 `updateStatus` 串行。
- filter 超长：段数 > `maxSegments` 或命令超长 → 写 `filter_complex_script` 文件传 `-filter_complex_script`。
- 不处理：区域级帧差、ASR、LLM、atempo 同步原音、任务中心接入（均 v2+）。

---

## 6. 下游依赖调用

```
common.media.FfmpegProbe#probe(Path): ProbeResult              — 时长/编码/fps
common.media.FfmpegProbe#isFfmpegAvailable(): boolean
common.media.FfmpegProcessRegistry#spawn(ProcessBuilder): Process — track + onExit untrack
common.sse.SseEmitterRegistry#create(key) / publish(key,event,payload) — per-job 进度频道
```

---

## 7. 异常处理要点

- FFmpeg 不可用 → `FfmpegUnavailableException`（503）。
- path/musicPath 为空或非常规文件 → `IllegalArgumentException` / `NoSuchFileException`（400/404，走 `GlobalExceptionHandler`）。
- render 时作业非 ANALYZED / 曲线非法（speed≤0、重叠）→ `IllegalArgumentException`（400）。
- ffmpeg 卡死/超时 → `waitFor` 超时 → `destroyForcibly` → job FAILED + stderr 尾部写 error。
- 取消已终态 → `IllegalStateException`（409）。
- 分析/渲染异常一律捕获 → job 置 FAILED 并 publish，绝不让虚拟线程静默吞掉。
```

