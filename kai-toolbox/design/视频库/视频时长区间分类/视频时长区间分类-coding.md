# 视频时长区间分类（编码摘要）

> 配套：[视频时长区间分类-current.md](视频时长区间分类-current.md) · [api-current.md](视频时长区间分类-api-current.md)

## 0. 一句话设计结论

按 size DESC 扫 `duration_s IS NULL` 的 video 行，调 `FfmpegProbe.probe()` 拿 duration，按 5 段切 bucket，写回 video 表。复用 `VideoProcessingJobService`，type=DURATION_PROBE。

## 1. 核心规则

- 顺序：`WHERE duration_s IS NULL ORDER BY size DESC`
- ffprobe 超时（yml 已配 5s）/ 文件缺失 → `duration_bucket='unknown'`
- 区间切分：< 30s / 30s-5min / 5min-30min / 30min-90min / > 90min / unknown
- 单 virtual thread + 礼让播放（与其他任务一致）

## 2. 接口入口

| 接口 | 实现 |
|------|------|
| `POST /duration-probe/start` | `TreeSizeController#startDurationProbe` → `VideoDurationProbeService#start` |
| `POST /duration-probe/stop` | `VideoDurationProbeService#stop` |
| `GET /duration-probe/status` | `VideoProcessingJobService#getLatest(DURATION_PROBE)` |
| `GET /duration-probe/events` | `VideoProcessingJobService#subscribe(DURATION_PROBE)` |

## 3. 涉及类清单

### 3.1 新建

#### `com.exceptioncoder.toolbox.treesize.domain.DurationBucket`

```java
public enum DurationBucket {
    MICRO("micro",     0,         30),
    SHORT("short",     30,        300),
    MEDIUM("medium",   300,       1800),
    LONG("long",       1800,      5400),
    XLONG("xlong",     5400,      Integer.MAX_VALUE),
    UNKNOWN("unknown", -1,        -1);

    private final String label;
    private final int lowSec, highSec;
    DurationBucket(String label, int lowSec, int highSec) {
        this.label = label; this.lowSec = lowSec; this.highSec = highSec;
    }
    public String label() { return label; }

    public static DurationBucket fromSeconds(double s) {
        if (s < 30)   return MICRO;
        if (s < 300)  return SHORT;
        if (s < 1800) return MEDIUM;
        if (s < 5400) return LONG;
        return XLONG;
    }
}
```

#### `com.exceptioncoder.toolbox.treesize.service.VideoDurationProbeService`

```java
@Service
public class VideoDurationProbeService {

    private final VideoProcessingJobService jobService;
    private final VideoTableRepository videoRepo;
    private final FfmpegProbe ffprobe;
    private final ActivePlaybackTracker playback;

    public Optional<String> start() {
        return jobService.startJob(DURATION_PROBE, this::workerLoop);
    }

    public void stop() { jobService.cancelJob(DURATION_PROBE); }

    private void workerLoop(JobContext ctx) {
        long total = videoRepo.countNeedingDuration();
        jobService.setTotal(ctx, total);
        while (!ctx.cancelled().get()) {
            List<VideoRow> batch = videoRepo.findNeedingDuration(50, 0);
            if (batch.isEmpty()) break;
            for (VideoRow v : batch) {
                if (ctx.cancelled().get()) break;
                waitForPlaybackQuiet();
                try {
                    Path p = Path.of(v.path());
                    if (!Files.isRegularFile(p)) {
                        videoRepo.updateDuration(v.path(), null, "unknown");
                        jobService.recordFailure(ctx, v.path(), "file_not_found");
                        continue;
                    }
                    ProbeResult r = ffprobe.probe(p);
                    double s = r.duration();
                    if (s <= 0) {
                        videoRepo.updateDuration(v.path(), null, "unknown");
                        jobService.recordFailure(ctx, v.path(), "probe_no_duration");
                        continue;
                    }
                    DurationBucket bucket = DurationBucket.fromSeconds(s);
                    videoRepo.updateDuration(v.path(), s, bucket.label());
                    jobService.recordSuccess(ctx, v.path());
                } catch (Exception e) {
                    videoRepo.updateDuration(v.path(), null, "unknown");
                    jobService.recordFailure(ctx, v.path(), summarize(e));
                }
            }
        }
    }
}
```

### 3.2 改造

#### `ProcessingJobType` 加值

```java
public enum ProcessingJobType {
    LANGUAGE_DETECT, THUMBNAIL_GRID, PERSON_AGE_DETECT,
    DURATION_PROBE,        // 新增
}
```

#### `VideoTableRepository`

```java
public long countNeedingDuration() {
    return jdbc.queryForObject(
        "SELECT COUNT(*) FROM treesize_video WHERE duration_s IS NULL", Long.class);
}

public List<VideoRow> findNeedingDuration(int limit, int offset) {
    return jdbc.query(
        "SELECT ... FROM treesize_video WHERE duration_s IS NULL " +
        "ORDER BY size DESC LIMIT ? OFFSET ?",
        VideoTableRepository::mapRow, limit, offset);
}

public void updateDuration(String path, Double durationS, String bucket) {
    jdbc.update(
        "UPDATE treesize_video SET duration_s=?, duration_bucket=? WHERE path=?",
        durationS, bucket, path);
}
```

#### `TreeSizeController`

加 4 个端点，与其他任务同构。

#### 前端

`DurationProbeButton.tsx` + `DurationProbeProgressPanel.tsx`，模式与其他任务按钮一致。

## 4. 数据结构

`treesize_video` 新增 `duration_bucket TEXT`（`duration_s REAL` 父表已预留）。partial index：

```sql
CREATE INDEX idx_video_duration_bucket ON treesize_video(duration_bucket);
CREATE INDEX idx_video_duration_null ON treesize_video(size DESC) WHERE duration_s IS NULL;
```

## 5. 约束

- 复用 yml `toolbox.ffmpeg.probe-timeout-ms: 5000`
- 区间切分硬编码（不进 yml；改动频率低）
- 单 virtual thread；与其他任务独立判定

## 6. 测试要点

- 正常视频：duration_s + bucket 落库
- 损坏文件：unknown + reason
- 文件不存在：unknown + file_not_found
- 取消即时
- 并行其他任务允许
- 重启动 409
