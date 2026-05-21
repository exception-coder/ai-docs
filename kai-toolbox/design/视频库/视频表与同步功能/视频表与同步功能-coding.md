# 视频表与同步功能（编码摘要）

> 配套：[视频表与同步功能-current.md](视频表与同步功能-current.md) · [api-current.md](视频表与同步功能-api-current.md)
> 编码门禁第二道：本文件存在即可开始写代码。

## 0. 一句话设计结论

新建 `treesize_video` 独立表（path PK，basic+media+language 字段一次建齐），新增 `POST /api/treesize/videos/sync` 把 `treesize_node` 中 ≥100KB 的视频用 `INSERT OR IGNORE` 同步过来，前端加同步按钮。本期不切换视频库列表数据源。

## 1. 核心业务规则

- **过滤**：`is_dir=0 AND ext IN (VideoExtensionsProperties) AND size >= 102400` AND 对应 scan 的 `status='COMPLETED'`
- **同步语义**：`INSERT OR IGNORE`（path 主键冲突即跳过；保护下期填入的 language/media 数据）
- **不更新 / 不删除**：已存在行任何字段都不动；node 表里被删的视频在 video 表里会变成 stale 行（本期不处理）
- **新插入行的字段值**：basic 从 node 拷贝；`source_scan_id` = node 行的 scan_id；`first_synced_at` = `last_synced_at` = `System.currentTimeMillis()`；media/language 列全 NULL
- **skippedTooSmall**：单独跑一次 COUNT 查询拿到，让用户感知噪音规模（不参与去重逻辑）
- **错误**：任何 SQLException 直接抛 → @ControllerAdvice 转 500；不做局部 swallow

## 2. 接口入口指针

| 接口 | 实现类#方法 |
|------|------------|
| `POST /api/treesize/videos/sync` | `TreeSizeController#syncVideos` → `VideoSyncService#sync` |

字段级契约见 [api-current.md](视频表与同步功能-api-current.md)，不在此重复。

## 3. 涉及类清单

### 3.1 新建类

#### `com.exceptioncoder.toolbox.treesize.api.dto.VideoSyncResult`

```java
public record VideoSyncResult(
    long scannedFromNode,
    long insertedNew,
    long skippedExisting,
    long skippedTooSmall,
    long elapsedMs
) {}
```

职责：接口出参 DTO。

---

#### `com.exceptioncoder.toolbox.treesize.domain.VideoRow`

```java
public record VideoRow(
    // ===== 标识 + basic（同步时写入）=====
    String path,
    String name,
    String parentPath,
    String ext,
    long size,
    String sourceScanId,
    long firstSyncedAt,
    long lastSyncedAt,
    // ===== media（duration_* 由时长分类模块；其它 NULL）=====
    Double durationS,
    String durationBucket,
    Integer width,
    Integer height,
    String videoCodec,
    String audioCodec,
    String audioLangTag,
    // ===== language（视频语言识别子模块）=====
    String language,
    Double languageConfidence,
    Long languageDetectedAt,
    // ===== thumbnail_grid（视频九宫格预览图子模块）=====
    String thumbnailGridPath,
    Long thumbnailGridGeneratedAt,
    // ===== person_age（视频人物年龄识别子模块）=====
    String personMainAgeGroup,
    Integer personMainAge,
    String personMainGender,
    Double personAgeConfidence,
    Long personAgeDetectedAt,
    String personAgeReason,
    // ===== series（视频名称归类子模块）=====
    String seriesSignature,
    Integer seriesEpisode,
    // ===== visual_cluster（视频嵌入与相似聚类子模块）=====
    Integer visualClusterId,
    String visualClusterLabel,
    Long visualClusteredAt
) {
    /** 同步场景用：仅 basic 字段，其它全 NULL。 */
    public static VideoRow forSync(String path, String name, String parentPath, String ext,
                                    long size, String sourceScanId, long now) {
        return new VideoRow(path, name, parentPath, ext, size, sourceScanId, now, now,
            // media
            null, null, null, null, null, null, null,
            // language
            null, null, null,
            // thumbnail_grid
            null, null,
            // person_age
            null, null, null, null, null, null,
            // series
            null, null,
            // visual_cluster
            null, null, null);
    }
}
```

职责：`treesize_video` 行的不可变映射。同步路径用 `forSync` 工厂；各子模块通过 Repository 的 `updateXxx(...)` 方法写入对应字段。

---

#### `com.exceptioncoder.toolbox.treesize.repository.VideoTableRepository`

```java
@Repository
public class VideoTableRepository {

    private final JdbcTemplate jdbc;

    public VideoTableRepository(JdbcTemplate jdbc) { this.jdbc = jdbc; }

    /** 本期同步入口：batch insert，path 冲突自动跳过。返回实际插入行数。 */
    public long batchInsertIgnore(List<VideoRow> rows) {
        if (rows.isEmpty()) return 0;
        int[] results = jdbc.batchUpdate(
            "INSERT OR IGNORE INTO treesize_video " +
            "(path, name, parent_path, ext, size, source_scan_id, " +
            " first_synced_at, last_synced_at) " +
            "VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
            new BatchPreparedStatementSetter() {
                @Override public void setValues(PreparedStatement ps, int i) throws SQLException {
                    VideoRow r = rows.get(i);
                    ps.setString(1, r.path());
                    ps.setString(2, r.name());
                    ps.setString(3, r.parentPath());
                    ps.setString(4, r.ext());
                    ps.setLong(5, r.size());
                    ps.setString(6, r.sourceScanId());
                    ps.setLong(7, r.firstSyncedAt());
                    ps.setLong(8, r.lastSyncedAt());
                }
                @Override public int getBatchSize() { return rows.size(); }
            });
        long inserted = 0;
        for (int r : results) if (r > 0) inserted += r;
        return inserted;
    }

    /** 查 treesize_video 当前行数（用于校验 / debug）。 */
    public long count() { ... }

    /** 子模块用：查待处理视频 path，按 size DESC（大文件优先）。 */
    public List<VideoRow> findNeedingLanguageDetect(int limit, int offset) { ... }
    public List<VideoRow> findNeedingThumbnailGrid(int limit, int offset) { ... }

    /** 子模块用：写入识别 / 生成结果。 */
    public void updateLanguage(String path, String iso, double confidence, long detectedAt) { ... }
    public void updateThumbnailGrid(String path, String gridPath, long generatedAt) { ... }
}
```

职责：`treesize_video` 表的所有 JDBC 读写。两个子模块所有针对 video 表的写都通过本类暴露的方法走，**禁止子模块自己拼 SQL**。

---

#### `com.exceptioncoder.toolbox.treesize.service.VideoSyncService`

```java
@Service
public class VideoSyncService {

    private static final long MIN_SIZE_BYTES = 100L * 1024;

    private final JdbcTemplate jdbc;
    private final VideoTableRepository videoRepo;
    private final VideoExtensionsProperties videoExt;

    // ctor 注入 ...

    public VideoSyncResult sync() {
        long t0 = System.currentTimeMillis();
        List<String> exts = videoExt.getExtensions().stream()
            .map(s -> s.toLowerCase(Locale.ROOT))
            .toList();

        // 1. 跑一次 COUNT 拿过滤掉的小文件数（用户感知噪音规模）
        long skippedTooSmall = countVideosBelowSize(exts);

        // 2. SELECT 出所有合格视频节点（scan COMPLETED + ext + size>=100KB）
        List<VideoRow> candidates = selectVideosFromNode(exts);

        long now = System.currentTimeMillis();
        List<VideoRow> withTimestamp = candidates.stream()
            .map(r -> VideoRow.forSync(r.path(), r.name(), r.parentPath(), r.ext(),
                                        r.size(), r.sourceScanId(), now))
            .toList();

        // 3. batch INSERT OR IGNORE
        long inserted = videoRepo.batchInsertIgnore(withTimestamp);
        long skippedExisting = withTimestamp.size() - inserted;

        return new VideoSyncResult(
            withTimestamp.size(),
            inserted,
            skippedExisting,
            skippedTooSmall,
            System.currentTimeMillis() - t0);
    }

    private long countVideosBelowSize(List<String> exts) {
        // SELECT COUNT(*) FROM treesize_node n
        // JOIN treesize_scan s ON n.scan_id=s.id
        // WHERE s.status='COMPLETED' AND n.is_dir=0
        //   AND lower(n.ext) IN (...) AND n.size < ?
    }

    private List<VideoRow> selectVideosFromNode(List<String> exts) {
        // SELECT n.path, n.name, n.parent_path, n.ext, n.size, n.scan_id
        // FROM treesize_node n
        // JOIN treesize_scan s ON n.scan_id=s.id
        // WHERE s.status='COMPLETED' AND n.is_dir=0
        //   AND lower(n.ext) IN (...) AND n.size >= ?
        // ORDER BY n.path  -- path 升序便于 INSERT OR IGNORE 命中顺序稳定
    }
}
```

职责：同步业务逻辑入口。**不直接执行 SQL DDL**（DDL 由 TreeSizeMigration 负责）。

---

### 3.2 改造类

#### `com.exceptioncoder.toolbox.treesize.repository.TreeSizeMigration`

新增两个 `ensure*` 方法，在 `run()` 里 `ensureExtColumn()` 之后调用：

```java
private void ensureVideoTable() {
    // 完整字段清单详见 current.md §5.1
    jdbc.execute("""
        CREATE TABLE IF NOT EXISTS treesize_video (
            path TEXT PRIMARY KEY,
            name TEXT NOT NULL,
            parent_path TEXT, ext TEXT, size INTEGER NOT NULL,
            source_scan_id TEXT,
            first_synced_at INTEGER NOT NULL, last_synced_at INTEGER NOT NULL,
            -- media
            duration_s REAL, duration_bucket TEXT,
            width INTEGER, height INTEGER,
            video_codec TEXT, audio_codec TEXT, audio_lang_tag TEXT,
            -- language
            language TEXT, language_confidence REAL, language_detected_at INTEGER,
            -- thumbnail_grid
            thumbnail_grid_path TEXT, thumbnail_grid_generated_at INTEGER,
            -- person_age
            person_main_age_group TEXT, person_main_age INTEGER, person_main_gender TEXT,
            person_age_confidence REAL, person_age_detected_at INTEGER, person_age_reason TEXT,
            -- series
            series_signature TEXT, series_episode INTEGER,
            -- visual_cluster
            visual_cluster_id INTEGER, visual_cluster_label TEXT, visual_clustered_at INTEGER
        )
        """);
    // 基础索引
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_size              ON treesize_video(size)");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_name              ON treesize_video(name COLLATE NOCASE)");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_ext               ON treesize_video(ext)");
    // 按字段过滤的索引
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_language          ON treesize_video(language)");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_duration_bucket   ON treesize_video(duration_bucket)");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_person_age_group  ON treesize_video(person_main_age_group)");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_series_signature  ON treesize_video(series_signature)");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_cluster           ON treesize_video(visual_cluster_id)");
    // partial index：各任务的"待处理列表"扫描
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_language_null    ON treesize_video(size DESC) WHERE language IS NULL");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_grid_null        ON treesize_video(size DESC) WHERE thumbnail_grid_path IS NULL");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_person_age_null  ON treesize_video(size DESC) WHERE thumbnail_grid_path IS NOT NULL AND person_main_age_group IS NULL");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_duration_null    ON treesize_video(size DESC) WHERE duration_s IS NULL");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_video_series_null      ON treesize_video(size DESC) WHERE series_signature IS NULL");
}

private void ensureProcessingJobTable() {
    jdbc.execute("""
        CREATE TABLE IF NOT EXISTS video_processing_job (
            id TEXT PRIMARY KEY,
            type TEXT NOT NULL,
            status TEXT NOT NULL,
            total INTEGER NOT NULL DEFAULT 0,
            processed INTEGER NOT NULL DEFAULT 0,
            succeeded INTEGER NOT NULL DEFAULT 0,
            failed INTEGER NOT NULL DEFAULT 0,
            current_path TEXT,
            error_msg TEXT,
            started_at INTEGER NOT NULL,
            finished_at INTEGER
        )
        """);
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_job_type_status ON video_processing_job(type, status)");
    jdbc.execute("CREATE INDEX IF NOT EXISTS idx_job_started     ON video_processing_job(started_at DESC)");
}
```

`run()` 调用顺序：

```java
private void run() {
    try {
        ensureExtColumn();
        ensureVideoTable();           // 新增
        ensureProcessingJobTable();   // 新增
        ensureVideoIndexes();
        ensureSubtitleColumns();
        backfillExt();
        ...
    } ...
}
```

> partial index 的 `WHERE language IS NULL` 子句让待处理列表的扫描永远不需要全表 —— 处理完一个视频，UPDATE 把 language 写入后该行自动从 partial index 中移除。九宫格同理。

#### `com.exceptioncoder.toolbox.treesize.api.TreeSizeController`

新增字段 + 端点：

```java
private final VideoSyncService videoSync;   // 构造注入

@PostMapping("/videos/sync")
public VideoSyncResult syncVideos() {
    return videoSync.sync();
}
```

#### `kai-toolbox/tools/tool-treesize/src/main/resources/db/treesize-schema.sql`

末尾 append（让全新部署的库一次性建表，避免依赖 migration）：

```sql
CREATE TABLE IF NOT EXISTS treesize_video ( ... 同 §5.1 ... );
CREATE INDEX ...（6 个索引同 §5.1）

CREATE TABLE IF NOT EXISTS video_processing_job ( ... 同 §5.2 ... );
CREATE INDEX ...（2 个索引同 §5.2）
```

#### 前端 `frontend/src/features/video-library/api.ts`

```typescript
export interface VideoSyncResult {
  scannedFromNode: number
  insertedNew: number
  skippedExisting: number
  skippedTooSmall: number
  elapsedMs: number
}

export function syncVideoLibrary() {
  return http<VideoSyncResult>('/treesize/videos/sync', { method: 'POST' })
}
```

#### 前端 `frontend/src/features/video-library/pages/VideoLibraryPage.tsx`

加入：

```typescript
const syncMutation = useMutation({
  mutationFn: syncVideoLibrary,
  onSuccess: async (result) => {
    await confirm({
      title: '同步完成',
      description: (
        <div className="space-y-1 text-sm">
          <div>扫描 <strong>{result.scannedFromNode}</strong> 个视频</div>
          <div>新增 <strong className="text-emerald-600">{result.insertedNew}</strong> 条</div>
          <div>已存在跳过 {result.skippedExisting} 条</div>
          {result.skippedTooSmall > 0 && (
            <div className="text-xs text-[var(--color-muted-foreground)]">
              （另过滤掉 {result.skippedTooSmall} 个 &lt;100KB 的噪音文件）
            </div>
          )}
          <div className="text-xs text-[var(--color-muted-foreground)]">耗时 {result.elapsedMs} ms</div>
        </div>
      ),
      confirmText: '知道了',
      cancelText: '关闭',
    })
  },
})

const handleSync = () => syncMutation.mutate()
```

并把 `onSync={handleSync}` / `syncing={syncMutation.isPending}` 传给 `VideoListPanel`。

#### 前端 `frontend/src/features/video-library/components/VideoListPanel.tsx`

`Props` 增加：

```typescript
onSync?: () => void
syncing?: boolean
```

顶栏在「清理 ._ 文件」按钮**左边**加：

```tsx
{onSync && (
  <button
    type="button"
    onClick={onSync}
    disabled={syncing}
    className="inline-flex items-center gap-1 whitespace-nowrap rounded-md border px-2 py-1.5 text-xs hover:bg-[var(--color-accent)] disabled:opacity-50"
    title="把扫描发现的视频同步到视频表（不存在才插入）"
  >
    {syncing ? <Loader2 className="h-3.5 w-3.5 animate-spin" /> : <RefreshCw className="h-3.5 w-3.5" />}
    同步视频库
  </button>
)}
```

`RefreshCw` 图标从 `lucide-react` import。

## 4. 数据结构（treesize_video）

完整字段含义见 [current.md §5.1](视频表与同步功能-current.md#51-新增表-treesize_video)。本期同步路径只会写以下列；其它列保持 NULL：

| 列 | 来源 | 本期写入 |
|---|------|---------|
| `path` | node.path | ✅ |
| `name` | node.name | ✅ |
| `parent_path` | node.parent_path | ✅ |
| `ext` | node.ext | ✅ |
| `size` | node.size | ✅ |
| `source_scan_id` | node.scan_id | ✅ |
| `first_synced_at` | `System.currentTimeMillis()` | ✅ |
| `last_synced_at` | `System.currentTimeMillis()` | ✅ |
| 其它 media/language 字段 | — | ❌ NULL（下期填） |

## 5. 重要约束与边界

| 约束 | 说明 |
|------|------|
| **只增不改** | 已存在 `path` 任何列都不动。已检测到的 language 字段不会被同步覆盖 |
| **SQL 过滤优先** | 100KB 阈值在 SQL `WHERE` 里过滤；Java 端不再二次判 |
| **不走 SSE** | 同步阻塞执行，前端按钮 disabled + Loading；万级 < 1s，可接受 |
| **不动 NodeRepository** | 视频库列表查询保持现状；不在本期切数据源 |
| **不引入新依赖** | 使用现有 `JdbcTemplate` / `VideoExtensionsProperties` |
| **路径分隔符** | 跨平台保持与 treesize_node.path 完全一致（Windows 反斜杠 / POSIX 斜杠不做归一化） |
| **大小写敏感** | `ext` 来自 node 表已小写，无需额外 lower；扩展名白名单 IN 子句要把 props 列表也 lower 化 |
| **事务** | 单事务 + batch insert；中途失败整体回滚（默认 Spring JDBC 行为） |

## 6. 测试要点

- **空表场景**：treesize_node 无视频时返回 `scannedFromNode=0 / insertedNew=0`，不报错
- **重复点击**：连点两次，第二次应 `insertedNew=0 / skippedExisting=N`
- **<100KB 过滤**：插入若干 50KB 的 .mp4 行到 treesize_node，确认不进 video 表，但出现在 `skippedTooSmall` 计数中
- **多 scan_id 同 path**：两个 scan 包含同一视频路径，确认只插一行，`source_scan_id` 取第一次出现的（与 `ORDER BY n.path` + IGNORE 行为对齐）
- **status != COMPLETED 的 scan**：FAILED / IN_PROGRESS 的 scan 里的视频不应进 video 表
- **大库性能**：treesize_node 含 10 万行（非视频 + 视频混合）时，同步 < 5s（开发机基准）
- **schema 迁移**：旧库（无 treesize_video 表）启动后 Migration 应自动建表 + 4 索引

## 7. 不在本期实现

| 项 | 推迟到 |
|---|--------|
| ffprobe 探 duration/width/height/codec/audio_lang_tag | 下期媒体属性模块 |
| Whisper -dl 检测 language | 下期语言识别模块 |
| 视频库列表数据源切换到 treesize_video | 下期视频库 v2 |
| 扫盘完成事件触发自动同步 | 下期事件接入 |
| 清理孤儿行（path 已从 node 表消失） | 下期清理工具 |
| subtitle_job.source_language 回填到 video 表 | 下期语言识别模块的副产物 |
