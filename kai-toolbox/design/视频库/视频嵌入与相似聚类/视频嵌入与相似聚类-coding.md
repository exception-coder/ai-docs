# 视频嵌入与相似聚类（编码摘要）

> 配套：[视频嵌入与相似聚类-current.md](视频嵌入与相似聚类-current.md) · [api-current.md](视频嵌入与相似聚类-api-current.md)

## 0. 一句话设计结论

九宫格 jpg 喂 ai-vision `/embedding/visual` (DINOv2) 拿 768-D 向量，BLOB 存到 `video_embedding` 表；后续相似检索全表余弦排序；自动聚类调 ai-vision `/clustering/hdbscan` 拿 label 写回 `treesize_video.visual_cluster_id`。

## 1. 核心规则

详见 current.md §5。要点：
- DINOv2-base 输出 768-D 已归一化
- 向量 BLOB = `float32 * 768 = 3072 bytes`
- 相似度 = 点积（向量已归一化时等价 cosine）
- HDBSCAN min_cluster_size=5
- noise label = -1

## 2. 接口入口

| 接口 | 实现 |
|------|------|
| `POST /visual/embed/{start,stop}` | `VideoVisualEmbedService` |
| `GET /visual/embed/{status,events}` | `VideoProcessingJobService#{getLatest,subscribe}(VISUAL_EMBED)` |
| `POST /visual/cluster/start` | `VideoVisualClusterService#start` |
| `GET /visual/cluster/status` | `VideoProcessingJobService#getLatest(VISUAL_CLUSTER)` |
| `GET /visual/similar` | `VisualSimilaritySearcher#findSimilar` |
| `GET /visual/clusters` | `VideoVisualClusterService#listClusters` |
| `GET /visual/cluster/{id}` | `VideoTableRepository#findByVisualClusterId` |

## 3. 涉及类清单

### 3.1 ai-vision 新建文件

#### `python-services/ai-vision/models/visual_embed.py`

```python
import io
import os
import logging
import numpy as np
import torch
from fastapi import FastAPI, UploadFile, File, HTTPException
from PIL import Image

log = logging.getLogger("ai-vision.visual_embed")
MODEL_NAME = "visual_embed"

def _load():
    from transformers import AutoModel, AutoImageProcessor
    repo = os.getenv("DINOV2_MODEL", "facebook/dinov2-base")
    log.info("loading DINOv2: %s", repo)
    processor = AutoImageProcessor.from_pretrained(repo)
    model = AutoModel.from_pretrained(repo).to("cuda").eval().half()
    return {"processor": processor, "model": model, "repo": repo}

def register(app: FastAPI, registry):
    registry.register_loader(MODEL_NAME, _load)

    @app.post("/embedding/visual")
    async def embed(file: UploadFile = File(...)):
        m = registry.get(MODEL_NAME)
        try:
            img = Image.open(io.BytesIO(await file.read())).convert("RGB")
        except Exception as e:
            raise HTTPException(400, f"invalid image: {e}")
        with torch.no_grad():
            inputs = m["processor"](images=img, return_tensors="pt").to("cuda")
            outputs = m["model"](**{k: v.half() if v.dtype==torch.float else v for k,v in inputs.items()})
            # DINOv2 用 CLS token embedding
            emb = outputs.last_hidden_state[:, 0, :].float().cpu().numpy()[0]
            # L2 归一化（让点积等价 cosine）
            emb = emb / (np.linalg.norm(emb) + 1e-8)
        return {
            "vector": emb.tolist(),
            "dim": int(emb.shape[0]),
            "model": m["repo"].split("/")[-1],
        }
```

#### `python-services/ai-vision/models/cluster.py`

```python
import logging
import numpy as np
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

log = logging.getLogger("ai-vision.cluster")
# 这个不是模型，是无状态函数，但还是走 registry 是为了一致
MODEL_NAME = "cluster"

def _load():
    # 仅导入校验
    import hdbscan  # noqa: F401
    return {"ready": True}

class ClusterRequest(BaseModel):
    vectors: list[list[float]]
    min_cluster_size: int = 5

def register(app: FastAPI, registry):
    registry.register_loader(MODEL_NAME, _load)

    @app.post("/clustering/hdbscan")
    def cluster(req: ClusterRequest):
        registry.get(MODEL_NAME)  # 触发加载校验
        import hdbscan
        if len(req.vectors) < 2:
            raise HTTPException(400, "need at least 2 vectors to cluster")
        X = np.array(req.vectors, dtype=np.float32)
        clusterer = hdbscan.HDBSCAN(min_cluster_size=req.min_cluster_size, metric="euclidean")
        labels = clusterer.fit_predict(X)
        return {
            "labels": labels.tolist(),
            "n_clusters": int(labels.max() + 1) if labels.max() >= 0 else 0,
        }
```

#### `requirements.txt` 补充

```text
transformers>=4.40.0
hdbscan>=0.8.33
```

### 3.2 Java 端新建

#### `com.exceptioncoder.toolbox.treesize.client.AiVisionClient` 加方法

```java
public EmbeddingResult computeEmbedding(byte[] imageBytes, String filename) {
    // 同 analyzeFaceAge 的 multipart
    return http.post().uri("/embedding/visual").body(...).retrieve().body(EmbeddingResult.class);
}

public ClusterResult cluster(List<float[]> vectors, int minClusterSize) {
    Map<String, Object> body = Map.of("vectors", vectors, "min_cluster_size", minClusterSize);
    return http.post()
        .uri("/clustering/hdbscan")
        .contentType(MediaType.APPLICATION_JSON)
        .body(body)
        .retrieve()
        .body(ClusterResult.class);
}
```

#### DTO

```java
public record EmbeddingResult(float[] vector, int dim, String model) {}
public record ClusterResult(int[] labels, int nClusters) {}
public record SimilarVideo(String path, String name, String thumbnailGridPath, double similarity) {}
```

#### `VideoEmbeddingRepository`

```java
@Repository
public class VideoEmbeddingRepository {
    private final JdbcTemplate jdbc;

    public void upsert(String path, String model, int dim, float[] vector, long generatedAt) {
        byte[] blob = floatArrayToBytes(vector);   // little-endian float32
        jdbc.update(
            "INSERT INTO video_embedding(path, model, dim, vector, generated_at) " +
            "VALUES(?,?,?,?,?) " +
            "ON CONFLICT(path) DO UPDATE SET " +
            "  model=excluded.model, dim=excluded.dim, " +
            "  vector=excluded.vector, generated_at=excluded.generated_at",
            path, model, dim, blob, generatedAt);
    }

    public float[] load(String path) { ... }

    /** 流式扫全表（用 RowMapper 边读边吐，避免一次性把 1万 * 3KB = 30MB 全装内存）。 */
    public Stream<EmbedRow> streamAll(String model) {
        return jdbc.queryForStream(
            "SELECT path, vector FROM video_embedding WHERE model=?",
            (rs, i) -> new EmbedRow(rs.getString(1), bytesToFloatArray(rs.getBytes(2))),
            model);
    }

    public long countNeedingEmbedding() {
        return jdbc.queryForObject(
            "SELECT COUNT(*) FROM treesize_video v " +
            "LEFT JOIN video_embedding e ON e.path=v.path " +
            "WHERE v.thumbnail_grid_path IS NOT NULL AND e.path IS NULL",
            Long.class);
    }

    public record EmbedRow(String path, float[] vector) {}

    private static byte[] floatArrayToBytes(float[] arr) {
        ByteBuffer buf = ByteBuffer.allocate(arr.length * 4).order(ByteOrder.LITTLE_ENDIAN);
        for (float f : arr) buf.putFloat(f);
        return buf.array();
    }
    private static float[] bytesToFloatArray(byte[] bytes) {
        FloatBuffer fb = ByteBuffer.wrap(bytes).order(ByteOrder.LITTLE_ENDIAN).asFloatBuffer();
        float[] arr = new float[fb.remaining()];
        fb.get(arr);
        return arr;
    }
}
```

#### `VideoVisualEmbedService`

模式与「视频人物年龄识别」完全一致，仅服务名 + 调用方法不同：

```java
public Optional<String> start() {
    if (!aiVision.isHealthy()) throw new IllegalStateException("ai-vision 未启动");
    return jobService.startJob(VISUAL_EMBED, this::workerLoop);
}

private void workerLoop(JobContext ctx) {
    long total = embeddingRepo.countNeedingEmbedding();
    jobService.setTotal(ctx, total);
    while (!ctx.cancelled().get()) {
        List<VideoRow> batch = videoRepo.findNeedingVisualEmbedding(50, 0);
        if (batch.isEmpty()) break;
        for (VideoRow v : batch) {
            if (ctx.cancelled().get()) break;
            try {
                Path g = Path.of(v.thumbnailGridPath());
                if (!Files.isRegularFile(g)) {
                    jobService.recordFailure(ctx, v.path(), "grid_missing");
                    continue;
                }
                byte[] bytes = Files.readAllBytes(g);
                EmbeddingResult r = aiVision.computeEmbedding(bytes, g.getFileName().toString());
                embeddingRepo.upsert(v.path(), r.model(), r.dim(), r.vector(),
                                      System.currentTimeMillis());
                jobService.recordSuccess(ctx, v.path());
            } catch (Exception e) {
                jobService.recordFailure(ctx, v.path(), summarize(e));
            }
        }
    }
}
```

#### `VideoVisualClusterService`

```java
public Optional<String> start() {
    if (!aiVision.isHealthy()) throw new IllegalStateException("ai-vision 未启动");
    return jobService.startJob(VISUAL_CLUSTER, this::run);
}

private void run(JobContext ctx) {
    // 1. 收集全部向量
    List<EmbeddingRepo.EmbedRow> rows;
    try (var s = embeddingRepo.streamAll("dinov2-base")) {
        rows = s.toList();
    }
    if (rows.size() < 5) {
        jobService.finish(ctx, FAILED, System.currentTimeMillis(), "embedding count < 5");
        return;
    }
    jobService.setTotal(ctx, rows.size());

    // 2. 调聚类
    List<float[]> vectors = rows.stream().map(EmbeddingRepo.EmbedRow::vector).toList();
    ClusterResult r;
    try { r = aiVision.cluster(vectors, 5); }
    catch (Exception e) {
        jobService.finish(ctx, FAILED, System.currentTimeMillis(), summarize(e));
        return;
    }

    // 3. 写回
    long now = System.currentTimeMillis();
    for (int i = 0; i < rows.size(); i++) {
        if (ctx.cancelled().get()) break;
        videoRepo.updateVisualCluster(rows.get(i).path(), r.labels()[i], null, now);
        jobService.recordSuccess(ctx, rows.get(i).path());
    }
}

public List<ClusterSummary> listClusters() { ... }   // SQL group by visual_cluster_id
```

#### `VisualSimilaritySearcher`

```java
@Component
public class VisualSimilaritySearcher {
    private final VideoEmbeddingRepository embedRepo;
    private final VideoTableRepository videoRepo;

    public List<SimilarVideo> findSimilar(String videoPath, int topN, double minSim) {
        float[] query = embedRepo.load(videoPath);
        if (query == null) throw new NotFoundException("no embedding for: " + videoPath);
        record Scored(String path, double sim) {}
        try (var s = embedRepo.streamAll("dinov2-base")) {
            List<Scored> scored = s
                .filter(r -> !r.path().equals(videoPath))
                .map(r -> new Scored(r.path(), dot(query, r.vector())))
                .filter(x -> x.sim() >= minSim)
                .sorted(Comparator.comparingDouble(Scored::sim).reversed())
                .limit(topN)
                .toList();
            // 二次查 video 表拿 name + grid path
            List<String> paths = scored.stream().map(Scored::path).toList();
            Map<String, VideoRow> rowMap = videoRepo.findByPaths(paths).stream()
                .collect(Collectors.toMap(VideoRow::path, x -> x));
            return scored.stream().map(s2 -> {
                VideoRow row = rowMap.get(s2.path());
                return new SimilarVideo(s2.path(), row.name(), row.thumbnailGridPath(), s2.sim());
            }).toList();
        }
    }

    private static double dot(float[] a, float[] b) {
        double sum = 0;
        for (int i = 0; i < a.length; i++) sum += a[i] * b[i];
        return sum;
    }
}
```

### 3.3 改造

#### `ProcessingJobType` 加值

```java
public enum ProcessingJobType {
    LANGUAGE_DETECT, THUMBNAIL_GRID, PERSON_AGE_DETECT,
    DURATION_PROBE, NAME_GROUPING,
    VISUAL_EMBED, VISUAL_CLUSTER,    // 新增
}
```

#### `VideoTableRepository`

```java
public List<VideoRow> findNeedingVisualEmbedding(int limit, int offset) {
    return jdbc.query(
        "SELECT v.* FROM treesize_video v " +
        "LEFT JOIN video_embedding e ON e.path=v.path " +
        "WHERE v.thumbnail_grid_path IS NOT NULL AND e.path IS NULL " +
        "ORDER BY v.size DESC LIMIT ? OFFSET ?",
        VideoTableRepository::mapRow, limit, offset);
}

public void updateVisualCluster(String path, int clusterId, String label, long clusteredAt) {
    jdbc.update(
        "UPDATE treesize_video SET visual_cluster_id=?, visual_cluster_label=?, " +
        "visual_clustered_at=? WHERE path=?",
        clusterId, label, clusteredAt, path);
}

public List<VideoRow> findByVisualClusterId(int clusterId) {
    return jdbc.query(
        "SELECT ... FROM treesize_video WHERE visual_cluster_id=? ORDER BY size DESC",
        VideoTableRepository::mapRow, clusterId);
}

public List<VideoRow> findByPaths(Collection<String> paths) { ... }
```

#### `TreeSizeController` — 加 8 个端点

按 §2 接口表逐一实现。

#### 前端

`VisualEmbedButton.tsx` + 进度面板，`VisualClusterButton.tsx`；视频详情页加"相似视频"侧栏（下期）；列表项按聚类分组视图（下期）。

## 4. 数据结构

新表 `video_embedding`：详见 current.md §3.5。

`treesize_video` 新增三字段：`visual_cluster_id INTEGER` / `visual_cluster_label TEXT` / `visual_clustered_at INTEGER`。

`TreeSizeMigration` 加 `ensureEmbeddingTable()` / `ensureVisualClusterColumns()`。

## 5. 重要约束

- 向量必须 L2 归一化后存（DINOv2 服务端已归一）
- BLOB 用 little-endian float32（约定）
- 聚类是 one-shot，启动即全集；不在线增量
- 相似检索是同步 HTTP，不进任务表
- 数据量 < 10 万时不上 sqlite-vec

## 6. 测试要点

- embedding 入库：向量长度 768、L2 范数 ≈ 1.0
- 相似检索：自己跟自己相似度 = 1.0（除查询时排除）
- 聚类：5 个明显不同的视频应分到不同簇
- ai-vision 未启动：start 拒绝
- 取消：CANCELLED；已生成的向量保留
- 流式扫描：内存峰值不爆（用 try-with-resources 关 Stream）
- BLOB 序列化往返：write → read 应得到完全相同的 float[]

## 7. 不在本期实现

详见 current.md §8。
