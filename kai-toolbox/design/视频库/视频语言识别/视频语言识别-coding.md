# 视频语言识别（编码摘要）

> 配套：[视频语言识别-current.md](视频语言识别-current.md) · [api-current.md](视频语言识别-api-current.md)

## 0. 一句话设计结论

按钮触发后台 virtual thread，按 `size DESC` 处理 `treesize_video` 中 `language IS NULL` 的行；每个视频 ffmpeg 抽 25% 位置 60s WAV → WhisperRunner.detectLanguage(`-dl` 模式) → 写回 video 表。任务用 `video_processing_job` 跟踪，SSE 推进度，礼让用户播放。

## 1. 核心业务规则

- **顺序**：`SELECT path FROM treesize_video WHERE language IS NULL ORDER BY size DESC`（partial index 加持）
- **抽样**：25% 偏移 + 60s 单声道 16kHz；视频 < 60s 用全长；< 5s 跳过标记 too_short
- **Whisper 模式**：`-dl`（detect-language）不输出 VTT
- **并发**：单 virtual thread 串行，GPU 串行最稳
- **礼让**：复用 `ActivePlaybackTracker.recentlyActive(15_000)`，与 ThumbnailWarmer 同源
- **取消**：worker 在循环顶端检查 `AtomicBoolean cancelled`，立即 break；正在运行的 Whisper 进程被 `destroyForcibly`
- **失败容错**：单视频失败计 `failed++` + `errorMsg` 写最近错误，不中断
- **致命错误**：Whisper 二进制缺失 / dry-run 失败 → 任务 FAILED 不开跑
- **同时只一个**：启动前 SELECT WHERE type='LANGUAGE_DETECT' AND status='RUNNING'，存在则 409

## 2. 接口入口指针

| 接口 | 实现类#方法 |
|------|------------|
| `POST /language-detect/start` | `TreeSizeController#startLanguageDetect` → `VideoLanguageDetectionService#start` |
| `POST /language-detect/stop` | `TreeSizeController#stopLanguageDetect` → `VideoLanguageDetectionService#stop` |
| `GET  /language-detect/status` | `TreeSizeController#getLanguageDetectStatus` → `VideoProcessingJobService#getLatest(LANGUAGE_DETECT)` |
| `GET  /language-detect/events` | `TreeSizeController#languageDetectEvents` → `VideoProcessingJobService#subscribe(LANGUAGE_DETECT)` |

字段级契约见 [api-current.md](视频语言识别-api-current.md)。

## 3. 涉及类清单

### 3.1 新建类

#### `com.exceptioncoder.toolbox.treesize.domain.ProcessingJobType`
```java
public enum ProcessingJobType { LANGUAGE_DETECT, THUMBNAIL_GRID }
```

#### `com.exceptioncoder.toolbox.treesize.domain.ProcessingJobStatus`
```java
public enum ProcessingJobStatus { RUNNING, DONE, FAILED, CANCELLED }
```

#### `com.exceptioncoder.toolbox.treesize.domain.ProcessingJob`
```java
public record ProcessingJob(
    String id, ProcessingJobType type, ProcessingJobStatus status,
    long total, long processed, long succeeded, long failed,
    String currentPath, String errorMsg,
    long startedAt, Long finishedAt
) {}
```

#### `com.exceptioncoder.toolbox.treesize.domain.DetectedLanguage`
```java
public record DetectedLanguage(String iso, double confidence) {}
```

#### `com.exceptioncoder.toolbox.treesize.repository.ProcessingJobRepository`
方法：
- `String insertRunning(ProcessingJobType type, long startedAt)` → 返回 jobId
- `void updateTotal(String jobId, long total)`
- `void updateProgress(String jobId, long processed, long succeeded, long failed, String currentPath, String errorMsg)`
- `void finish(String jobId, ProcessingJobStatus status, long finishedAt)`
- `Optional<ProcessingJob> findRunning(ProcessingJobType type)`
- `Optional<ProcessingJob> findLatest(ProcessingJobType type)`
- `void cleanupStaleRunning()` 启动时清理：把 `status=RUNNING AND finished_at IS NULL` 的行置为 FAILED（错误信息 "interrupted by restart"）

#### `com.exceptioncoder.toolbox.treesize.service.VideoProcessingJobService`
共用任务调度抽象。

字段：
```java
private final ProcessingJobRepository jobRepo;
private final Map<ProcessingJobType, JobHandle> running = new ConcurrentHashMap<>();
private final Map<ProcessingJobType, Set<SseEmitter>> emitters = new ConcurrentHashMap<>();

record JobHandle(String jobId, AtomicBoolean cancelled, Thread thread) {}
```

方法：
```java
/** 原子启动：若已有 RUNNING 同类型任务，返回 Optional.empty()；否则建 job 行 + 起 virtual thread。 */
public Optional<String> startJob(ProcessingJobType type, Consumer<JobContext> workerLoop);
public void cancelJob(ProcessingJobType type);
public Optional<ProcessingJob> getLatest(ProcessingJobType type);
public SseEmitter subscribe(ProcessingJobType type);    // 标准 SseEmitter 注册

/** worker 调用：上报进度 + 广播 SSE event=progress */
public void recordSuccess(JobContext ctx, String path);
public void recordFailure(JobContext ctx, String path, String errorMsg);
public void setTotal(JobContext ctx, long total);
```

`JobContext` record：`String jobId, AtomicBoolean cancelled, ProcessingJobType type`。

#### `com.exceptioncoder.toolbox.treesize.service.VideoLanguageDetectionService`

```java
@Service
public class VideoLanguageDetectionService {

    private static final double SAMPLE_START_PERCENT = 0.25;
    private static final double SAMPLE_DURATION_S = 60.0;
    private static final double MIN_VIDEO_DURATION_S = 5.0;

    private final VideoProcessingJobService jobService;
    private final VideoTableRepository videoRepo;
    private final WhisperRunner whisper;
    private final FfmpegProcessRegistry ffmpeg;
    private final FfmpegProbe ffprobe;
    private final ActivePlaybackTracker playback;
    private final Path tmpDir;   // {java.io.tmpdir}/kai-toolbox/lang-detect

    public Optional<String> start() {
        // dry-run whisper -dl 一次
        try { dryRunWhisper(); }
        catch (Exception e) { throw new IllegalStateException("whisper -dl unavailable: " + e); }
        return jobService.startJob(LANGUAGE_DETECT, this::workerLoop);
    }

    public void stop() { jobService.cancelJob(LANGUAGE_DETECT); }

    private void workerLoop(JobContext ctx) {
        long total = videoRepo.countNeedingLanguageDetect();
        jobService.setTotal(ctx, total);
        int offset = 0;
        while (!ctx.cancelled().get()) {
            List<VideoRow> batch = videoRepo.findNeedingLanguageDetect(50, 0);  // partial index 自动跳过已处理
            if (batch.isEmpty()) break;
            for (VideoRow v : batch) {
                if (ctx.cancelled().get()) break;
                waitForPlaybackQuiet();
                Path wav = null;
                try {
                    double durationS = resolveDuration(v);
                    if (durationS < MIN_VIDEO_DURATION_S) {
                        jobService.recordFailure(ctx, v.path(), "too_short");
                        continue;
                    }
                    double startSec = Math.max(0, durationS * SAMPLE_START_PERCENT);
                    double sampleDur = Math.min(SAMPLE_DURATION_S, durationS - startSec);
                    wav = tmpDir.resolve(UUID.randomUUID() + ".wav");
                    ffmpeg.extractAudioSlice(Path.of(v.path()), startSec, sampleDur, wav);
                    DetectedLanguage dl = whisper.detectLanguage(wav, ctx.cancelled());
                    long now = System.currentTimeMillis();
                    videoRepo.updateLanguage(v.path(), dl.iso(), dl.confidence(), now);
                    jobService.recordSuccess(ctx, v.path());
                } catch (Exception e) {
                    jobService.recordFailure(ctx, v.path(), summarize(e));
                } finally {
                    if (wav != null) Files.deleteIfExists(wav);
                }
            }
        }
    }

    private double resolveDuration(VideoRow v) {
        if (v.durationS() != null) return v.durationS();
        ProbeResult p = ffprobe.probe(Path.of(v.path()));
        return p.duration();
    }

    private void waitForPlaybackQuiet() {
        while (playback.recentlyActive(15_000)) {
            try { Thread.sleep(2_000); }
            catch (InterruptedException ie) { Thread.currentThread().interrupt(); return; }
        }
    }
}
```

### 3.2 改造类

#### `WhisperRunner` —— 加方法

```java
/** detect-language-only 模式：解码音频前段，仅返回检测出的 ISO 码 + 置信度。 */
public DetectedLanguage detectLanguage(Path wav, AtomicBoolean cancelled) throws IOException, InterruptedException {
    if (!props.isAvailable()) throw new IllegalStateException("Whisper 不可用");
    List<String> cmd = new ArrayList<>();
    cmd.add(props.getBinary());
    cmd.add(props.getCli().getModelFlag()); cmd.add(props.getModelPath());
    cmd.add(props.getCli().getFileFlag());  cmd.add(wav.toAbsolutePath().toString());
    cmd.add("--detect-language");           // whisper.cpp 标准 flag
    if (props.isDisableGpu()) cmd.add(props.getCli().getNoGpuFlag());
    else if (props.isFlashAttention()) cmd.add(props.getCli().getFlashAttnFlag());

    Process p = registry.spawn(new ProcessBuilder(cmd).redirectErrorStream(true));
    StringBuilder out = new StringBuilder();
    // 复用现有 stderr reader 模式 + LANGUAGE_RE
    var reader = Thread.ofVirtual().start(() -> readAll(p, out));
    waitForExit(p, cancelled);
    reader.join(1000);

    Matcher m = LANGUAGE_RE_WITH_P.matcher(out);
    if (!m.find()) throw new IOException("whisper -dl: language not parsed from output");
    return new DetectedLanguage(m.group(1), Double.parseDouble(m.group(2)));
}

// 新正则：捕获 ISO 码 + 概率
private static final Pattern LANGUAGE_RE_WITH_P =
    Pattern.compile("auto-detected language:\\s*([a-zA-Z\\-]+)\\s*\\(p\\s*=\\s*(\\d+(?:\\.\\d+)?)\\)");
```

#### `FfmpegProcessRegistry`（或邻近类，看现有结构选）—— 加方法

```java
/** 切音频片段为 whisper 兼容的 wav。 */
public void extractAudioSlice(Path src, double startSec, double durationSec, Path outWav)
        throws IOException, InterruptedException {
    Files.createDirectories(outWav.getParent());
    List<String> cmd = List.of(
        props.getBinary(),
        "-y",
        "-ss", String.valueOf(startSec),
        "-t",  String.valueOf(durationSec),
        "-i",  src.toAbsolutePath().toString(),
        "-vn",
        "-ac", "1",
        "-ar", "16000",
        outWav.toAbsolutePath().toString()
    );
    Process p = spawn(new ProcessBuilder(cmd).redirectErrorStream(true));
    if (!p.waitFor(120, TimeUnit.SECONDS)) {
        p.destroyForcibly();
        throw new IOException("ffmpeg extract audio timeout");
    }
    if (p.exitValue() != 0) throw new IOException("ffmpeg exit " + p.exitValue());
}
```

#### `VideoTableRepository` —— 加方法

```java
public long countNeedingLanguageDetect() {
    return jdbc.queryForObject(
        "SELECT COUNT(*) FROM treesize_video WHERE language IS NULL", Long.class);
}

public List<VideoRow> findNeedingLanguageDetect(int limit, int offset) {
    return jdbc.query(
        "SELECT path, name, parent_path, ext, size, source_scan_id, " +
        "       first_synced_at, last_synced_at, duration_s, width, height, " +
        "       video_codec, audio_codec, audio_lang_tag, " +
        "       language, language_confidence, language_detected_at, " +
        "       thumbnail_grid_path, thumbnail_grid_generated_at " +
        "FROM treesize_video WHERE language IS NULL " +
        "ORDER BY size DESC LIMIT ? OFFSET ?",
        VideoTableRepository::mapRow, limit, offset);
}

public void updateLanguage(String path, String iso, double confidence, long detectedAt) {
    jdbc.update(
        "UPDATE treesize_video SET language=?, language_confidence=?, language_detected_at=? WHERE path=?",
        iso, confidence, detectedAt, path);
}
```

#### `TreeSizeController` —— 加端点

```java
private final VideoLanguageDetectionService langSvc;
private final VideoProcessingJobService jobSvc;

@PostMapping("/videos/language-detect/start")
public ResponseEntity<LanguageDetectStartResult> startLanguageDetect() {
    Optional<String> jobId = langSvc.start();
    if (jobId.isEmpty()) {
        ProcessingJob running = jobSvc.getLatest(LANGUAGE_DETECT).orElseThrow();
        return ResponseEntity.status(409).body(
            new LanguageDetectStartResult(running.id(), running.total(), "already running"));
    }
    long total = jobSvc.getLatest(LANGUAGE_DETECT).map(ProcessingJob::total).orElse(0L);
    return ResponseEntity.ok(new LanguageDetectStartResult(jobId.get(), total, null));
}

@PostMapping("/videos/language-detect/stop")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void stopLanguageDetect() { langSvc.stop(); }

@GetMapping("/videos/language-detect/status")
public ProcessingJob getLanguageDetectStatus() {
    return jobSvc.getLatest(LANGUAGE_DETECT).orElse(null);
}

@GetMapping("/videos/language-detect/events")
public SseEmitter languageDetectEvents() {
    return jobSvc.subscribe(LANGUAGE_DETECT);
}
```

#### 前端 `api.ts` —— 见 api-current.md §调用示例

#### 前端 `VideoListPanel.tsx` —— 顶栏加按钮

```tsx
{onStartLanguage && (
  <LanguageDetectButton
    onStart={onStartLanguage}
    onStop={onStopLanguage}
    job={languageJob}
  />
)}
```

#### 前端 `LanguageDetectButton.tsx`

```tsx
// 自带 useQuery 拉 status + useEventSource 订阅 events
// idle / running 两种态：
// - idle: 显示「识别语言」按钮，点击 → onStart
// - running: 显示进度 (123/1842) + 「停止」按钮
```

## 4. 数据结构

- `treesize_video.language` / `language_confidence` / `language_detected_at`：详见父文档 §5.1
- `video_processing_job` 行：详见父文档 §5.2

## 5. 重要约束与边界

| 约束 | 说明 |
|------|------|
| **GPU 串行** | 单线程 worker，不并发 Whisper |
| **不抢用户** | recentlyActive(15s) 礼让 |
| **存量不重跑** | partial index `WHERE language IS NULL` 自动只挑没识别过的 |
| **WAV 临时清理** | finally 块删；任务结束统一清空目录（启动时也清一次防残留） |
| **取消即时** | cancelled.set(true) → worker 循环顶端 break；正在跑的 whisper destroyForcibly |
| **错误信息长度** | error_msg 列截断到 500 字符（防 Whisper 异常栈塞爆表） |
| **dry-run** | 启动前用一个内置的 1s 静音 wav（或现有测试样本）跑一次 `-dl`，失败则直接拒绝启动 |

## 6. 测试要点

- 启动 + 完成全套：插入 5 个 video 行，启动任务，等待 DONE，验证 language 字段全部填上
- 部分失败：插入 1 个不存在路径的 video（其它路径正常），任务跑完应 succeeded=4 / failed=1，state=DONE
- 取消：长任务跑到一半点 stop，CANCELLED；已识别的行 language 保留
- 重复启动：两次 start 应第二次返回 409
- 视频 < 5s：标记 failed=too_short，不抽 wav 不调 whisper
- Whisper 不可用：dry-run 失败 → /start 返回 500，不建 job 行
- 礼让：播放期间触发任务，worker 应在 sleep 中等待

## 7. 不在本期实现

| 项 | 推迟到 |
|---|--------|
| 多片段投票（低置信度） | 下期 |
| audio_lang_tag 优先快速通道 | 下期媒体属性模块联动 |
| 重新识别 UI | 下期 |
| subtitle_job.source_language 双向同步 | 下期 |
