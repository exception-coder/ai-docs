# 视频名称归类（编码摘要）

> 配套：[视频名称归类-current.md](视频名称归类-current.md) · [api-current.md](视频名称归类-api-current.md)

## 0. 一句话设计结论

`NameNormalizer.normalize(name) → (signature, episode)`：内置正则按顺序去噪（字幕组/画质/编码/年份/分辨率/语言标签等）→ 抽集数 → 标点归一 → 小写。结果写回 video 表 `series_signature` / `series_episode`。无 AI 依赖。

## 1. 核心规则

详见 current.md §4。

## 2. 接口入口

| 接口 | 实现 |
|------|------|
| `POST /name-grouping/start` | `TreeSizeController#startNameGrouping` → `VideoNameGroupingService#start` |
| `POST /name-grouping/stop` | `VideoNameGroupingService#stop` |
| `GET /name-grouping/status` | `VideoProcessingJobService#getLatest(NAME_GROUPING)` |
| `GET /name-grouping/events` | `VideoProcessingJobService#subscribe(NAME_GROUPING)` |
| `GET /videos/series/{sig}` | `TreeSizeController#getVideoSeries` → `VideoTableRepository#findBySeriesSignature` |

## 3. 涉及类清单

### 3.1 新建

#### `com.exceptioncoder.toolbox.treesize.service.NameNormalizer`

```java
@Component
public class NameNormalizer {

    private final NameGroupingProperties props;

    public NameNormalizer(NameGroupingProperties props) { this.props = props; }

    /** 入口：原始文件名（含扩展名） → 归一化签名 + 集数。 */
    public NormalizedName normalize(String filename) {
        String s = stripExtension(filename);

        // 1. 提取集数（在去噪前做，避免数字被一并删掉）
        Integer episode = extractEpisode(s);

        // 2. 按顺序应用去噪规则
        s = stripBracketTags(s);          // [VCB] 【喵萌】
        s = stripYear(s);                  // (2024) 【2024】
        s = stripQualityTags(s);           // 1080p 4K x265 BDrip ...
        s = stripResolution(s);            // 1920x1080
        s = stripLanguageTags(s);          // 中字 简中 双语
        s = stripReleaseGroupSuffix(s);    // -ABC 末尾连字符短语
        s = stripEpisodeMarkers(s);        // E01 第01话 #01 等
        s = stripEmptyBrackets(s);         // [] () 【】
        s = normalizePunctuation(s);       // . _ - → 空格；连续空格 → 一个；trim
        s = s.toLowerCase(Locale.ROOT);

        if (s.isBlank()) s = stripExtension(filename).toLowerCase(Locale.ROOT);

        return new NormalizedName(s, episode);
    }

    private String stripExtension(String name) { ... }
    private String stripBracketTags(String s) {
        // 删 [xxx]、【xxx】、(xxx) 这些方括号包裹的字幕组/压制组标签
        return s.replaceAll("\\[[^\\]]*\\]", "")
                .replaceAll("【[^】]*】", "");
    }
    private String stripYear(String s) {
        return s.replaceAll("\\(\\s*20\\d{2}\\s*\\)", "")
                .replaceAll("\\s20\\d{2}\\s", " ");
    }
    private static final Pattern QUALITY_RE = Pattern.compile(
        "(?i)\\b(1080p|720p|480p|2160p|4k|hdr|sdr|bdrip|webrip|web-dl|bluray|bd|" +
        "x264|x265|h\\.?264|h\\.?265|hevc|avc|aac|flac|ac3|dts|10bit|hi10p|" +
        "remux|repack|proper)\\b");
    private String stripQualityTags(String s) {
        return QUALITY_RE.matcher(s).replaceAll(" ");
    }
    private static final Pattern RESOLUTION_RE = Pattern.compile("\\d{3,4}[xX*]\\d{3,4}");
    private String stripResolution(String s) { return RESOLUTION_RE.matcher(s).replaceAll(" "); }
    private static final Pattern LANG_RE = Pattern.compile(
        "(中字|简中|繁中|双语|内嵌|外挂|英字|日字|中日双语|国语|粤语|英语|日语)");
    private String stripLanguageTags(String s) { return LANG_RE.matcher(s).replaceAll(" "); }
    private String stripReleaseGroupSuffix(String s) {
        // 保守：仅当末尾形如 "-ABC" 或 "-ABCDEF" 时去掉
        return s.replaceAll("\\s*-\\s*[A-Za-z0-9]{2,8}\\s*$", " ");
    }
    private static final Pattern EPISODE_MARKER_RE = Pattern.compile(
        "(?i)(?:e|ep|第)\\s*\\d{1,4}\\s*(?:话|集)?|#\\d{1,4}|第[一二三四五六七八九十百千]+[话集]");
    private String stripEpisodeMarkers(String s) { return EPISODE_MARKER_RE.matcher(s).replaceAll(" "); }
    private String stripEmptyBrackets(String s) {
        return s.replaceAll("\\[\\s*\\]", "")
                .replaceAll("\\(\\s*\\)", "")
                .replaceAll("【\\s*】", "");
    }
    private String normalizePunctuation(String s) {
        s = s.replace('.', ' ').replace('_', ' ').replace('-', ' ');
        s = s.replaceAll("\\s+", " ").trim();
        return s;
    }

    private static final Pattern EPISODE_NUM_RE = Pattern.compile(
        "(?i)(?:e|ep|第)\\s*(\\d{1,4})\\s*(?:话|集)?|#(\\d{1,4})|\\[(\\d{1,4})\\]");
    private static final Pattern EPISODE_CN_RE = Pattern.compile("第([一二三四五六七八九十百千]+)[话集]");

    public Integer extractEpisode(String s) {
        var m = EPISODE_NUM_RE.matcher(s);
        if (m.find()) {
            for (int i = 1; i <= 3; i++) {
                if (m.group(i) != null) {
                    try { return Integer.parseInt(m.group(i)); }
                    catch (NumberFormatException ignored) {}
                }
            }
        }
        var m2 = EPISODE_CN_RE.matcher(s);
        if (m2.find()) return chineseToArabic(m2.group(1));
        return null;
    }

    /** 中文数字串 → 阿拉伯数字。上限 props.chineseNumeralMax。 */
    public static int chineseToArabic(String cn) { ... }

    public record NormalizedName(String signature, Integer episode) {}
}
```

#### `com.exceptioncoder.toolbox.treesize.service.VideoNameGroupingService`

```java
@Service
public class VideoNameGroupingService {

    private final VideoProcessingJobService jobService;
    private final VideoTableRepository videoRepo;
    private final NameNormalizer normalizer;

    public Optional<String> start() {
        return jobService.startJob(NAME_GROUPING, this::workerLoop);
    }
    public void stop() { jobService.cancelJob(NAME_GROUPING); }

    private void workerLoop(JobContext ctx) {
        long total = videoRepo.countNeedingNameGrouping();
        jobService.setTotal(ctx, total);
        while (!ctx.cancelled().get()) {
            List<VideoRow> batch = videoRepo.findNeedingNameGrouping(200, 0);  // 处理快，batch 大点
            if (batch.isEmpty()) break;
            for (VideoRow v : batch) {
                if (ctx.cancelled().get()) break;
                try {
                    var n = normalizer.normalize(v.name());
                    videoRepo.updateSeries(v.path(), n.signature(), n.episode());
                    jobService.recordSuccess(ctx, v.path());
                } catch (Exception e) {
                    jobService.recordFailure(ctx, v.path(), summarize(e));
                }
            }
        }
    }
}
```

#### `com.exceptioncoder.toolbox.treesize.config.NameGroupingProperties`

```java
@ConfigurationProperties(prefix = "toolbox.name-grouping")
@Component
public class NameGroupingProperties {
    private List<String> extraNoisePatterns = List.of();
    private int chineseNumeralMax = 9999;
    // getter / setter
}
```

### 3.2 改造

#### `ProcessingJobType` 加值

```java
public enum ProcessingJobType {
    LANGUAGE_DETECT, THUMBNAIL_GRID, PERSON_AGE_DETECT,
    DURATION_PROBE, NAME_GROUPING,    // 新增
}
```

#### `VideoTableRepository`

```java
public long countNeedingNameGrouping() {
    return jdbc.queryForObject(
        "SELECT COUNT(*) FROM treesize_video WHERE series_signature IS NULL", Long.class);
}

public List<VideoRow> findNeedingNameGrouping(int limit, int offset) {
    return jdbc.query(
        "SELECT ... FROM treesize_video WHERE series_signature IS NULL " +
        "ORDER BY size DESC LIMIT ? OFFSET ?",
        VideoTableRepository::mapRow, limit, offset);
}

public void updateSeries(String path, String signature, Integer episode) {
    jdbc.update(
        "UPDATE treesize_video SET series_signature=?, series_episode=? WHERE path=?",
        signature, episode, path);
}

public List<VideoRow> findBySeriesSignature(String signature) {
    return jdbc.query(
        "SELECT ... FROM treesize_video WHERE series_signature=? " +
        "ORDER BY series_episode IS NULL, series_episode, name COLLATE NOCASE",
        VideoTableRepository::mapRow, signature);
}
```

#### `TreeSizeController`

加 5 个端点（4 个标准任务端点 + 1 个 `GET /videos/series/{sig}`）。

#### 前端

`NameGroupingButton.tsx` + 进度面板，与其他任务按钮同构。视频库列表项可在 series_signature 旁显示"同系列(N)"链接（下期再做）。

## 4. 数据结构

新增：

```sql
ALTER TABLE treesize_video ADD COLUMN series_signature TEXT;
ALTER TABLE treesize_video ADD COLUMN series_episode INTEGER;
CREATE INDEX idx_video_series_signature ON treesize_video(series_signature);
CREATE INDEX idx_video_series_null ON treesize_video(size DESC) WHERE series_signature IS NULL;
```

## 5. 重要约束

- 纯字符串处理，无外部依赖
- 单 virtual thread + 礼让（与其他任务一致，统一模式）
- batch 大小 200（处理快，可以激进）
- 规则可由 yml 扩展，但默认规则集已覆盖绝大多数场景

## 6. 测试要点

- 字幕组方括号去除：`[VCB] X.mp4` → signature=`x`
- 多种集数标记识别：`第01话` / `E01` / `EP01` / `[01]` / `#01` / `第一集` 都能抽出 episode
- 同系列不同集：相同 signature
- 不同压制版本：相同 signature（去画质标签后）
- 集数中文 → 阿拉伯：`第十五集` → 15
- 不带集数的电影：episode=null，signature=片名
- 极端命名：fallback 是原文件名 lowercase
- 重启动 409

## 7. 不在本期实现

详见 current.md §8。
