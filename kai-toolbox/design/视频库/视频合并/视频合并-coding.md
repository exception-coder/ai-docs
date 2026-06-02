# 视频合并 — 编码摘要

> 配套 `视频合并-current.md`。完整模版的编码落点，给出全类名 + 方法签名 + 关键实现约束。

## 0. 一句话

视频库多选 → `POST /api/treesize/videos/merge` 同步阻塞 → `VideoMergeService` 用 ffmpeg（copy 或 reencode）把 N 个视频拼成一个 mp4，落 merged 目录，返回计数。不进任务框架、不上 SSE、不回填视频表。

## 1. 核心业务规则（实现约束）

- 顺序 = 传入 `paths` 顺序。
- ffprobe 过滤：不存在 / 无视频流 / probe 失败 → skip 计数；有效 < 2 → `IllegalArgumentException`（全局转 400）。
- 策略：`auto` 看 `(videoCodec,audioCodec,container)` 一致性；`copy` / `force` 显式覆盖。
- 输出 `{outputDir}/merged_{mergedCount}clips_{System.currentTimeMillis()}.mp4`，失败删半成品。
- `maxInputs` / `timeoutS` 兜底。

## 2. 接口入口指针

| 方法 | 路径 | 实现 |
|---|---|---|
| 合并 | `POST /api/treesize/videos/merge` | `com.exceptioncoder.toolbox.treesize.api.VideoProcessingController#mergeVideos(VideoMergeRequest)` |

入参 `VideoMergeRequest{ List<String> paths; String reencode; }`；出参 `VideoMergeResult{ String outputPath; int inputCount; int mergedCount; int skippedCount; long outputBytes; boolean reencoded; long elapsedMs; }`。

## 3. 涉及类清单（全路径）

**新增：**
- `com.exceptioncoder.toolbox.treesize.service.VideoMergeService`
  - `public VideoMergeResult merge(VideoMergeRequest req)` — 编排主流程
  - `private List<Probed> probeAndFilter(List<String> paths)` — ffprobe + 过滤，`Probed{Path path; ProbeResult probe;}`
  - `private boolean uniform(List<Probed> inputs)` — 三元组一致性
  - `private Path resolveOutput(int mergedCount)` — 拼输出路径 + mkdirs
- `com.exceptioncoder.toolbox.treesize.api.dto.VideoMergeRequest`（record）
- `com.exceptioncoder.toolbox.treesize.api.dto.VideoMergeResult`（record）
- `com.exceptioncoder.toolbox.treesize.config.VideoMergeProperties`（`@ConfigurationProperties("toolbox.video.merge")`，需在 starter 的 `@ConfigurationPropertiesScan` 或 `@EnableConfigurationProperties` 范围内——参照 `VideoExtensionsProperties` 注册方式）

**改：**
- `com.exceptioncoder.toolbox.common.media.FfmpegProcessRegistry`
  - `public void concatCopy(List<Path> inputs, Path out, int timeoutS)` — 写 list.txt + `-f concat -safe 0 -c copy`
  - `public void concatReencode(List<Path> inputs, Path out, String resolution, int fps, boolean withAudio, int timeoutS)` — filter_complex scale+pad+fps+concat，软编 libx264(+aac)。`withAudio=false` 时输出无音轨（本期含无音轨输入的混合场景统一丢音轨，不做逐段补静音；后续可加 anullsrc+asplit）
  - `private void runConcat(List<String> cmd, Path out, int timeoutS, String threadName)` — copy/reencode 共用的执行壳（spawn+tail+超时强杀+退出码/空文件校验）
  - 复用现有 `spawn` / `startTailDrain` / `tailToString` / `waitFor+exitValue` 失败诊断；list.txt 写临时文件用后即删
- `com.exceptioncoder.toolbox.treesize.api.VideoProcessingController` — 注入 `VideoMergeService`，加 `@PostMapping("/merge")`
- `application.yml` — `toolbox.video.merge.{output-dir,target-resolution,target-fps,max-inputs,timeout-s}`

**前端：**
- `frontend/src/features/video-library/api.ts` — `mergeVideos(paths: string[])` + `VideoMergeResult` 接口
- `frontend/src/features/video-library/components/VideoListPanel.tsx` — 新增 prop `onBulkMerge?(items): Promise<void>`，多选工具栏加「合并」按钮（沿用 `selectedItems`，禁用条件 `selectedPaths.size < 2`）
- `frontend/src/features/video-library/pages/VideoLibraryPage.tsx` — 接 `onBulkMerge`，调 `mergeVideos`，`confirm` 弹窗展示 outputPath + mergedCount/skippedCount

## 4. 重要约束与边界

- ffmpeg list.txt 的 `file '...'` 路径要用绝对路径，单引号内含单引号需转义（碎片文件名一般无，仍做 `'\''` 转义防御）。
- reencode 路径：每个输入都过 scale+pad+setsar+fps。**音轨策略由 service 的 `allHaveAudio` 决定**：全部有音轨 → `concat=n=N:v=1:a=1` 保留音轨；只要有一个无音轨 → `concat=...:v=1:a=0` 整体输出无音轨（本期不做逐段 `anullsrc` 补静音，避免 a 流数量对不上报错）。
- 硬超时 destroyForcibly 后要 `descendants().forEach(destroyForcibly)`（与现有 extractAudioSlice 一致）。
- 输出非空校验：`Files.size(out) > 0`，否则当失败删文件。

## 5. 测试要点

- 同编码同分辨率 3 个 mp4 → auto 命中 copy，秒级，无损。
- 混合编码/分辨率 → auto 走 reencode，输出可播放、时长 ≈ 各输入之和。
- 含 1 个无音轨输入 → reencode 补静音成功；含 1 个损坏文件 → skip 计数 +1，其余正常合并。
- 选 1 个 → 400 too_few_valid_inputs。
- 超 maxInputs → 400。
