# 视频九宫格预览图（编码摘要）

> 配套：[视频九宫格预览图-current.md](视频九宫格预览图-current.md) · [api-current.md](视频九宫格预览图-api-current.md)

## 0. 一句话设计结论

按钮触发后台 virtual thread，按 `size DESC` 处理 `treesize_video` 中 `thumbnail_grid_path IS NULL` 的行；每个视频用单条 ffmpeg 命令（`fps + scale + pad + tile=3x3`）一次性出图，写到 `{thumb.cache-dir}/grid/{sha256(path)[0:16]}.jpg`，回写 video 表。复用 `VideoProcessingJobService` 任务调度。

## 1. 核心业务规则

- **顺序**：`SELECT path FROM treesize_video WHERE thumbnail_grid_path IS NULL ORDER BY size DESC`（partial index）
- **网格**：3×3 = 9 帧，固定
- **单帧**：320×180，保持原始 aspect ratio + pad 居中（竖屏视频两侧黑边）
- **抽帧位置**：时间均匀分布 9 个采样点，偏移半格避开首尾
- **输出格式**：JPEG q:v 4（≈ q=92），每张 30-80KB
- **输出路径**：`{toolbox.thumbnail.cache-dir}/grid/{sha256(absolutePath)[0:16]}.jpg`
- **缓存命中**：outPath 已存在但 video 表字段为 NULL → 不重跑 ffmpeg，直接 UPDATE
- **并发**：单 virtual thread；与语言识别独立判定（允许并行）
- **礼让**：复用 `ActivePlaybackTracker.recentlyActive(15_000)`
- **取消**：worker 检查 cancelled，立即 break；正在跑的 ffmpeg `destroyForcibly`
- **失败容错**：单视频失败计 `failed++`，继续
- **致命错误**：ffmpeg 不可用 / cache 目录不可写 → 任务 FAILED

## 2. 接口入口指针

| 接口 | 实现类#方法 |
|------|------------|
| `POST /thumbnail-grid/start` | `TreeSizeController#startThumbnailGrid` → `VideoThumbnailGridService#start` |
| `POST /thumbnail-grid/stop` | `TreeSizeController#stopThumbnailGrid` → `VideoThumbnailGridService#stop` |
| `GET  /thumbnail-grid/status` | `TreeSizeController#getThumbnailGridStatus` → `VideoProcessingJobService#getLatest(THUMBNAIL_GRID)` |
| `GET  /thumbnail-grid/events` | `TreeSizeController#thumbnailGridEvents` → `VideoProcessingJobService#subscribe(THUMBNAIL_GRID)` |
| `GET  /thumbnail-grid?path=` | `TreeSizeController#getThumbnailGrid` → `VideoThumbnailGridService#openGridStream(path)` |

字段级契约见 [api-current.md](视频九宫格预览图-api-current.md)。

## 3. 涉及类清单

### 3.1 共用基础设施

已由 [视频语言识别-coding.md §3.1](../视频语言识别/视频语言识别-coding.md#31-新建类) 落地：

- `ProcessingJobType` / `ProcessingJobStatus` / `ProcessingJob` / `ProcessingJobRepository` / `VideoProcessingJobService`

本模块**直接复用**，不再重复实现。

### 3.2 新建类

#### `com.exceptioncoder.toolbox.treesize.service.VideoThumbnailGridService`

```java
@Service
public class VideoThumbnailGridService {

    private static final int GRID_COLS = 3;
    private static final int GRID_ROWS = 3;
    private static final int CELL_W = 320;
    private static final int CELL_H = 180;
    private static final double MIN_DURATION_S = 2.0;
    private static final int FFMPEG_TIMEOUT_S = 120;

    private final VideoProcessingJobService jobService;
    private final VideoTableRepository videoRepo;
    private final FfmpegProcessRegistry ffmpeg;
    private final FfmpegProbe ffprobe;
    private final ActivePlaybackTracker playback;
    private final Path gridCacheDir;   // {thumb.cache-dir}/grid

    @PostConstruct
    public void init() throws IOException { Files.createDirectories(gridCacheDir); }

    public Optional<String> start() {
        if (!ffmpegProperties.isFfmpegAvailable()) throw new IllegalStateException("ffmpeg unavailable");
        if (!Files.isWritable(gridCacheDir)) throw new IllegalStateException("grid cache dir not writable");
        return jobService.startJob(THUMBNAIL_GRID, this::workerLoop);
    }

    public void stop() { jobService.cancelJob(THUMBNAIL_GRID); }

    private void workerLoop(JobContext ctx) {
        long total = videoRepo.countNeedingThumbnailGrid();
        jobService.setTotal(ctx, total);
        while (!ctx.cancelled().get()) {
            List<VideoRow> batch = videoRepo.findNeedingThumbnailGrid(50, 0);
            if (batch.isEmpty()) break;
            for (VideoRow v : batch) {
                if (ctx.cancelled().get()) break;
                waitForPlaybackQuiet();
                try {
                    Path src = Path.of(v.path());
                    if (!Files.isRegularFile(src)) {
                        jobService.recordFailure(ctx, v.path(), "file_not_found");
                        continue;
                    }
                    Path outPath = gridPathFor(v.path());
                    if (Files.exists(outPath)) {
                        // 缓存已有但库未回填 → 直接 UPDATE 不重跑 ffmpeg
                        videoRepo.updateThumbnailGrid(v.path(), outPath.toString(), System.currentTimeMillis());
                        jobService.recordSuccess(ctx, v.path());
                        continue;
                    }
                    double durationS = ffprobe.probe(src).duration();
                    if (durationS < MIN_DURATION_S) {
                        jobService.recordFailure(ctx, v.path(), "too_short");
                        continue;
                    }
                    ffmpeg.makeContactSheet(src, durationS, GRID_COLS, GRID_ROWS,
                                            CELL_W, CELL_H, outPath, FFMPEG_TIMEOUT_S);
                    videoRepo.updateThumbnailGrid(v.path(), outPath.toString(), System.currentTimeMillis());
                    jobService.recordSuccess(ctx, v.path());
                } catch (Exception e) {
                    jobService.recordFailure(ctx, v.path(), summarize(e));
                }
            }
        }
    }

    public InputStream openGridStream(String videoPath) throws IOException {
        // Controller 取图端点用：查库拿 grid_path，校验文件存在，返回输入流
        Optional<String> gridPath = videoRepo.findGridPathByVideoPath(videoPath);
        if (gridPath.isEmpty()) throw new NoSuchFileException("not generated");
        Path p = Path.of(gridPath.get());
        if (!Files.isRegularFile(p)) throw new NoSuchFileException("cache file gone");
        return Files.newInputStream(p);
    }

    private Path gridPathFor(String videoAbsPath) {
        String hash = sha256(videoAbsPath).substring(0, 16);
        return gridCacheDir.resolve(hash + ".jpg");
    }

    private void waitForPlaybackQuiet() {
        while (playback.recentlyActive(15_000)) {
            try { Thread.sleep(2_000); }
            catch (InterruptedException ie) { Thread.currentThread().interrupt(); return; }
        }
    }
}
```

### 3.3 改造类

#### `FfmpegProcessRegistry`（或邻近类）—— 加方法

```java
/**
 * Generate a contact sheet (NxM grid) from a video using a single ffmpeg invocation.
 *
 * @param src       source video
 * @param durationS source duration (from prior ffprobe)
 * @param cols      grid columns (e.g. 3)
 * @param rows      grid rows (e.g. 3)
 * @param cellW     each cell width (e.g. 320)
 * @param cellH     each cell height (e.g. 180)
 * @param outJpg    output JPEG path
 * @param timeoutS  hard timeout
 */
public void makeContactSheet(Path src, double durationS, int cols, int rows,
                              int cellW, int cellH, Path outJpg, int timeoutS)
        throws IOException, InterruptedException {
    Files.createDirectories(outJpg.getParent());
    int n = cols * rows;
    // 均匀抽 n 帧：fps = n/duration（每 duration/n 秒一帧），select=lt(...) 偏半格避首
    // scale 保 aspect ratio + pad 居中，避免拉伸
    String vf = String.format(Locale.ROOT,
        "fps=%d/%.3f,scale=%d:%d:force_original_aspect_ratio=decrease,"
        + "pad=%d:%d:(ow-iw)/2:(oh-ih)/2:color=black,tile=%dx%d",
        n, durationS, cellW, cellH, cellW, cellH, cols, rows);

    List<String> cmd = List.of(
        props.getBinary(),
        "-y",
        "-i", src.toAbsolutePath().toString(),
        "-vf", vf,
        "-frames:v", "1",
        "-q:v", "4",
        outJpg.toAbsolutePath().toString()
    );
    Process p = spawn(new ProcessBuilder(cmd).redirectErrorStream(true));
    if (!p.waitFor(timeoutS, TimeUnit.SECONDS)) {
        p.destroyForcibly();
        throw new IOException("contact sheet timeout for " + src);
    }
    if (p.exitValue() != 0) {
        throw new IOException("ffmpeg exit " + p.exitValue() + " for " + src);
    }
    if (!Files.isRegularFile(outJpg) || Files.size(outJpg) == 0) {
        throw new IOException("contact sheet produced empty output for " + src);
    }
}
```

#### `VideoTableRepository` —— 加方法

```java
public long countNeedingThumbnailGrid() {
    return jdbc.queryForObject(
        "SELECT COUNT(*) FROM treesize_video WHERE thumbnail_grid_path IS NULL", Long.class);
}

public List<VideoRow> findNeedingThumbnailGrid(int limit, int offset) {
    return jdbc.query(
        "SELECT ... FROM treesize_video WHERE thumbnail_grid_path IS NULL " +
        "ORDER BY size DESC LIMIT ? OFFSET ?",
        VideoTableRepository::mapRow, limit, offset);
}

public void updateThumbnailGrid(String path, String gridPath, long generatedAt) {
    jdbc.update(
        "UPDATE treesize_video SET thumbnail_grid_path=?, thumbnail_grid_generated_at=? WHERE path=?",
        gridPath, generatedAt, path);
}

public Optional<String> findGridPathByVideoPath(String videoPath) {
    List<String> r = jdbc.queryForList(
        "SELECT thumbnail_grid_path FROM treesize_video WHERE path=? AND thumbnail_grid_path IS NOT NULL",
        String.class, videoPath);
    return r.isEmpty() ? Optional.empty() : Optional.of(r.get(0));
}
```

#### `TreeSizeController` —— 加端点

```java
private final VideoThumbnailGridService gridSvc;

@PostMapping("/videos/thumbnail-grid/start")
public ResponseEntity<?> startThumbnailGrid() { /* 同语言识别 start 形态 */ }

@PostMapping("/videos/thumbnail-grid/stop")
@ResponseStatus(HttpStatus.NO_CONTENT)
public void stopThumbnailGrid() { gridSvc.stop(); }

@GetMapping("/videos/thumbnail-grid/status")
public ProcessingJob getThumbnailGridStatus() {
    return jobSvc.getLatest(THUMBNAIL_GRID).orElse(null);
}

@GetMapping("/videos/thumbnail-grid/events")
public SseEmitter thumbnailGridEvents() { return jobSvc.subscribe(THUMBNAIL_GRID); }

@GetMapping(value = "/videos/thumbnail-grid", produces = MediaType.IMAGE_JPEG_VALUE)
public ResponseEntity<StreamingResponseBody> getThumbnailGrid(@RequestParam String path) {
    try {
        InputStream in = gridSvc.openGridStream(path);
        StreamingResponseBody body = out -> { try (in) { in.transferTo(out); } };
        return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(Duration.ofDays(1)))
            .body(body);
    } catch (NoSuchFileException e) {
        return ResponseEntity.notFound().build();
    } catch (IOException e) {
        return ResponseEntity.status(500).build();
    }
}
```

#### 前端 `api.ts` —— 见 api-current.md §调用示例

#### 前端 `ThumbnailGridButton.tsx` / `ThumbnailGridProgressPanel.tsx`

与语言识别同结构，仅 API 路径不同。

#### 前端 `VideoListPanel.tsx` —— 顶栏

三个按钮并列：「同步视频库」「识别语言」「九宫格」。

## 4. 数据结构

- `treesize_video.thumbnail_grid_path` / `thumbnail_grid_generated_at`：见父文档 §5.1
- `video_processing_job`：见父文档 §5.2

## 5. 重要约束与边界

| 约束 | 说明 |
|------|------|
| **单 ffmpeg 命令出图** | 不存中间帧；ffmpeg 的 vf 链一次完成抽 + scale + pad + tile |
| **均匀抽样** | `fps=N/duration` 等价时间均匀；避免 select 复杂表达式 |
| **超时硬限** | 120s 单文件；超时 destroyForcibly |
| **缓存目录** | `{thumb.cache-dir}/grid/`，启动时确保存在；子模块不管 LRU |
| **路径哈希** | `sha256(absPath)[0:16]`，避免特殊字符 / 长度问题 |
| **缓存命中跳过** | outPath 已存在时跳过 ffmpeg，直接 UPDATE 字段 |
| **路径必须绝对** | repo 里存的就是绝对路径，与 treesize_video.path 完全一致 |
| **JPEG quality** | q:v=4（≈ q=92），平衡体积与画质 |
| **取图端点也走 video 表查 grid_path** | 不直接计算 hash 路径，避免迁移后失效 |

## 6. 测试要点

- 全套：3 个 video → 启动 → 等 DONE → 验证 3 个 jpg 存在 + 库字段填写
- 缓存命中：手动把 jpg 放到缓存目录但库字段 NULL → 跑任务应不调 ffmpeg 直接回填字段
- 取消：长任务跑到一半 stop → CANCELLED；已生成的 jpg 保留
- 取图：grid_path 为 NULL → 404；缓存文件被删 → 404
- 太短视频：1s 视频 → failed=too_short
- 文件不存在：插入一个假路径的 video → failed=file_not_found
- 并行任务：同时启动 LANGUAGE_DETECT + THUMBNAIL_GRID 应都允许 RUNNING
- 重复启动：两次 thumbnail-grid/start 应第二次 409

## 7. 不在本期实现

| 项 | 推迟到 |
|---|--------|
| 列表项 hover 显示九宫格 | 下期 |
| 自动检测缓存被删并重跑 | 下期 |
| 重新生成 UI | 下期 |
| 配置 cols × rows | 下期 |
| LRU 清理 | 下期 |
