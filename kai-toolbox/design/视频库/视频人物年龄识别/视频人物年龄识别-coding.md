# 视频人物年龄识别（编码摘要）

> 配套：[视频人物年龄识别-current.md](视频人物年龄识别-current.md) · [api-current.md](视频人物年龄识别-api-current.md)

## 0. 一句话设计结论

读 video 表里九宫格已生成但人物年龄未识别的行（`thumbnail_grid_path IS NOT NULL AND person_main_age_group IS NULL`），按 size DESC，把九宫格 jpg POST 到 ai-vision `/face-age/analyze`，拿回主体的 `{age_group, age, gender, confidence}` 写回 video 表。复用 `VideoProcessingJobService` 任务调度。

## 1. 核心业务规则

详见 current.md §5；要点：
- 输入九宫格 jpg；不重新抽帧
- 主体 = 最大 bbox 的人物
- 无人物 → `person_main_age_group='no_person'` + reason='no_person'
- 服务未启动 → 启动前抛清晰错误；运行中 60s health 缓存
- 失败容错；取消即时

## 2. 接口入口

| 接口 | 实现类#方法 |
|------|------------|
| `POST /person-age/start` | `TreeSizeController#startPersonAgeDetect` → `VideoPersonAgeService#start` |
| `POST /person-age/stop` | `TreeSizeController#stopPersonAgeDetect` → `VideoPersonAgeService#stop` |
| `GET /person-age/status` | `TreeSizeController#getPersonAgeStatus` → `VideoProcessingJobService#getLatest(PERSON_AGE_DETECT)` |
| `GET /person-age/events` | `TreeSizeController#personAgeEvents` → `VideoProcessingJobService#subscribe(PERSON_AGE_DETECT)` |

## 3. 涉及类清单

### 3.1 共用基础设施（已由其他子模块落地，本模块复用）

- `ProcessingJobType` 加枚举值 `PERSON_AGE_DETECT`
- `VideoProcessingJobService` / `ProcessingJobRepository` / `ProcessingJob` 复用
- `AiVisionClient` / `AiVisionProperties` / `FaceAgeResult` 由 Python AI 服务架构落地，本模块直接 @Autowired

### 3.2 新建类

#### `com.exceptioncoder.toolbox.treesize.domain.PersonAgeGroup`

```java
public enum PersonAgeGroup {
    NO_PERSON("no_person"),
    INFANT("infant"),
    CHILD("child"),
    TEEN("teen"),
    YOUNG_ADULT("young_adult"),
    MIDDLE_AGE("middle_age"),
    SENIOR("senior");

    private final String label;
    PersonAgeGroup(String label) { this.label = label; }
    public String label() { return label; }

    public static PersonAgeGroup parse(String s) {
        for (var v : values()) if (v.label.equals(s)) return v;
        throw new IllegalArgumentException("unknown age group: " + s);
    }
}
```

#### `com.exceptioncoder.toolbox.treesize.service.VideoPersonAgeService`

```java
@Service
public class VideoPersonAgeService {

    private static final Logger log = LoggerFactory.getLogger(VideoPersonAgeService.class);

    private final VideoProcessingJobService jobService;
    private final VideoTableRepository videoRepo;
    private final AiVisionClient aiVision;

    public Optional<String> start() {
        if (!aiVision.isHealthy()) {
            throw new IllegalStateException(
                "ai-vision 服务未启动，请双击 python-services/ai-vision/start.bat");
        }
        return jobService.startJob(PERSON_AGE_DETECT, this::workerLoop);
    }

    public void stop() { jobService.cancelJob(PERSON_AGE_DETECT); }

    private void workerLoop(JobContext ctx) {
        long total = videoRepo.countNeedingPersonAge();
        jobService.setTotal(ctx, total);
        while (!ctx.cancelled().get()) {
            List<VideoRow> batch = videoRepo.findNeedingPersonAge(50, 0);
            if (batch.isEmpty()) break;
            for (VideoRow v : batch) {
                if (ctx.cancelled().get()) break;
                try {
                    Path gridPath = Path.of(v.thumbnailGridPath());
                    if (!Files.isRegularFile(gridPath)) {
                        jobService.recordFailure(ctx, v.path(), "grid_missing");
                        continue;
                    }
                    byte[] bytes = Files.readAllBytes(gridPath);
                    FaceAgeResult r = aiVision.analyzeFaceAge(bytes, gridPath.getFileName().toString());
                    long now = System.currentTimeMillis();
                    if (r.mainPerson() == null) {
                        videoRepo.updatePersonAge(v.path(),
                            "no_person", null, null, null, now, "no_person");
                    } else {
                        var mp = r.mainPerson();
                        videoRepo.updatePersonAge(v.path(),
                            mp.ageGroup(), mp.age(), mp.gender(), mp.confidence(), now, null);
                    }
                    jobService.recordSuccess(ctx, v.path());
                } catch (Exception e) {
                    jobService.recordFailure(ctx, v.path(), summarize(e));
                }
            }
        }
    }

    private static String summarize(Throwable e) {
        String m = e.getClass().getSimpleName() + ": " + (e.getMessage() == null ? "" : e.getMessage());
        return m.length() > 500 ? m.substring(0, 500) : m;
    }
}
```

### 3.3 改造类

#### `ProcessingJobType` — 加枚举值

```java
public enum ProcessingJobType {
    LANGUAGE_DETECT,
    THUMBNAIL_GRID,
    PERSON_AGE_DETECT,    // 新增
}
```

#### `VideoTableRepository` — 加方法

```java
public long countNeedingPersonAge() {
    return jdbc.queryForObject(
        "SELECT COUNT(*) FROM treesize_video " +
        "WHERE thumbnail_grid_path IS NOT NULL AND person_main_age_group IS NULL",
        Long.class);
}

public List<VideoRow> findNeedingPersonAge(int limit, int offset) {
    return jdbc.query(
        "SELECT ... FROM treesize_video " +
        "WHERE thumbnail_grid_path IS NOT NULL AND person_main_age_group IS NULL " +
        "ORDER BY size DESC LIMIT ? OFFSET ?",
        VideoTableRepository::mapRow, limit, offset);
}

public void updatePersonAge(String path, String ageGroup, Integer age, String gender,
                             Double confidence, long detectedAt, String reason) {
    jdbc.update(
        "UPDATE treesize_video SET " +
        "  person_main_age_group=?, person_main_age=?, person_main_gender=?, " +
        "  person_age_confidence=?, person_age_detected_at=?, person_age_reason=? " +
        "WHERE path=?",
        ageGroup, age, gender, confidence, detectedAt, reason, path);
}
```

#### `TreeSizeController` — 加端点

与「视频语言识别」端点同构，path 改 `/person-age/*`，service 改 `personAgeSvc`。略。

#### 前端

与语言识别 / 九宫格按钮同构；`PersonAgeDetectButton.tsx` + `PersonAgeDetectProgressPanel.tsx` 各 60-100 行；`VideoListPanel.tsx` 顶栏现在四个按钮：同步 / 识别语言 / 九宫格 / 识别人物年龄。

## 4. 数据结构

`treesize_video` 加 6 个字段（详见父表文档修订后的 §5.1）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `person_main_age_group` | TEXT | 枚举 7 值（含 no_person） |
| `person_main_age` | INTEGER | 主体年龄数值，no_person 时为 NULL |
| `person_main_gender` | TEXT | `M`/`F`/`unknown`/NULL |
| `person_age_confidence` | REAL | 0..1，no_person 时为 NULL |
| `person_age_detected_at` | INTEGER | epoch ms |
| `person_age_reason` | TEXT | NULL（正常）/ `no_person` / `low_confidence` / `inference_failed` |

partial index：`CREATE INDEX idx_video_person_age_null ON treesize_video(size DESC) WHERE thumbnail_grid_path IS NOT NULL AND person_main_age_group IS NULL`。

## 5. 重要约束与边界

| 约束 | 说明 |
|------|------|
| 前置 | 九宫格已生成才进入待处理列表 |
| 主体选取 | 最大 bbox，规则简单稳定 |
| 服务依赖 | ai-vision 必须运行；启动前 isHealthy() 校验 |
| 健康缓存 | AiVisionClient 内置 60s，避免每图都打 |
| HTTP timeout | 60s 单请求；MiVOLO 单图实测 < 2s，余量充足 |
| no_person 落库 | 写明确值避免下次任务重跑 |

## 6. 测试要点

- 正常视频：识别后字段填齐
- 风景视频：no_person 标记落库
- 九宫格被删：grid_missing 失败计数
- ai-vision 未启动：start 拒绝执行，错误消息含启动指引
- 取消：CANCELLED；已识别行保留
- 并行：与语言识别 / 九宫格同时跑应都允许 RUNNING
- 重复启动：第二次 409

## 7. 不在本期实现

详见 current.md §8。
