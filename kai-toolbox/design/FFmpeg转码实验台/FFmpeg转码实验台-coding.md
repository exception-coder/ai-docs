# FFmpeg 转码实验台 编码摘要

> 对应设计：[FFmpeg转码实验台-current.md](FFmpeg转码实验台-current.md) ｜ 接口：[FFmpeg转码实验台-api-current.md](FFmpeg转码实验台-api-current.md)

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-05-31 | 初始版本 |

---

## 1. 核心业务规则

- 命令唯一源：`/probe` 预览命令必须等于 `/run` 实跑命令，均由 `ModeCommandBuilder` 产出。
- Remux 预判：仅 `canCopyVideo`(h264) ∧ `canCopyAudio`(aac/mp3/none) 才 OK，否则 FAIL。
- `clipSeconds>0` → 所有模式加 `-t`；0=整片。
- 进程一律走 `FfmpegProcessRegistry.spawn`；stderr 独立线程 drain 保留尾部 N 行；失败/断开强杀。
- ffmpeg 退出码非 0 仍返回 HTTP 200 + `success=false`（失败是实验台要呈现的结果）；唯 ffmpeg 不可用→503、文件不存在→404。
- NATIVE 类成功判定额外探产物：exit0+产物存在后再 `probe(out.mp4)`，`nativelyPlayable=false` 则 `success=false`（封装成功≠能播）。HLS=exit0+playlist；MJPEG=字节数>0。
- 无 SQLite；诊断走内存环（仿 PlaybackStatsCollector）；每 runId 一个 workDir 子目录，运行前清理过期目录。
- 临时文件类 spawn 时 `ProcessBuilder.directory(runDir)`：HLS fMP4 init 段裸名相对 CWD 写出，不设 cwd 会落到 app 启动目录 → init 段 404 / fragLoadError。

---

## 2. 接口入口指针

> 字段级契约见 api-current.md。

| 接口 | 实现类 #方法 |
|------|-------------|
| `GET /api/ffmpeg-lab/probe` | `FfmpegLabController#probe` |
| `POST /api/ffmpeg-lab/run` | `FfmpegLabController#run` |
| `GET /api/ffmpeg-lab/play/{mode}` | `FfmpegLabController#play` |
| `GET /api/ffmpeg-lab/runs/recent` | `FfmpegLabController#recentRuns` |

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.toolbox.ffmpeglab.domain.TranscodeMode` | 新建 | 5 模式枚举 + label + playKind + container |
| `com.exceptioncoder.toolbox.ffmpeglab.domain.RunResult` | 新建 | record 诊断记录 |
| `com.exceptioncoder.toolbox.ffmpeglab.domain.ModePrediction` | 新建 | enum OK/TRANSCODE/FAIL |
| `com.exceptioncoder.toolbox.ffmpeglab.service.ModeCommandBuilder` | 新建 | 命令构建唯一源 |
| `com.exceptioncoder.toolbox.ffmpeglab.service.FfmpegLabService` | 新建 | 探测/预判/运行编排 |
| `com.exceptioncoder.toolbox.ffmpeglab.service.RunDiagnosticsCollector` | 新建 | 内存环形缓冲 |
| `com.exceptioncoder.toolbox.ffmpeglab.config.FfmpegLabProperties` | 新建 | `toolbox.ffmpeg-lab.*` workDir/clipDefault/retainMinutes |
| `com.exceptioncoder.toolbox.ffmpeglab.config.FfmpegLabToolDescriptor` | 新建 | ToolDescriptor 注册 |
| `com.exceptioncoder.toolbox.ffmpeglab.api.FfmpegLabController` | 新建 | REST 入口 |
| `...api.dto.ProbeView / ModeView / RunRequest / RunResultView / RecentRunsView` | 新建 | DTO |

### 关键方法签名与职责

```
// 命令构建（唯一源）
ModeCommandBuilder#build(TranscodeMode mode, Path input, ProbeResult probe, int clipSeconds, Path workDir): List<String>
  — 产出 ffmpeg 参数列表（含 binary、-t clip、编码/封装参数、输出路径或 pipe:1）
ModeCommandBuilder#preview(...): String — 同 build 结果 join 成展示用命令串（输入/输出用占位）
ModeCommandBuilder#videoEncoder(): String — 按 FfmpegProperties.hwaccel 映射 libx264/h264_nvenc...（与 HlsService 同款）

// 编排
FfmpegLabService#probeAndPredict(String path, int clipSeconds): ProbeView
  — ffprobe + 对每 mode 算 ModePrediction + preview 命令
FfmpegLabService#run(RunRequest req): RunResultView
  — 临时文件类：spawn→waitFor→drain tail→校验产物→record→返回；流式(MJPEG)：返回 streaming=true+playUrl
FfmpegLabService#streamFile(String mode, String runId, OutputStream/Range...): void — 托管 workDir 产物（mp4/m3u8/分段）
FfmpegLabService#streamMjpeg(Path file, int clipSeconds, OutputStream out): void — spawn mpjpeg pipe 直出 + record
FfmpegLabService#predict(TranscodeMode mode, ProbeResult probe): ModePrediction
FfmpegLabService#cleanStaleWorkDirs(): void — 运行前清理过期 runId 目录

// 诊断环
RunDiagnosticsCollector#record(RunResult r): void
RunDiagnosticsCollector#recent(): List<RunResult>   // 倒序，上限 ~50
```

---

## 4. 数据结构

```java
// TranscodeMode 枚举
REMUX_COPY("Remux 直封装", PlayKind.NATIVE, "mp4")
PROGRESSIVE_MP4("Progressive MP4 全转码", PlayKind.NATIVE, "mp4")
HLS_TS("HLS (MPEG-TS)", PlayKind.HLS, "m3u8")
HLS_FMP4("HLS (fMP4/CMAF)", PlayKind.HLS, "m3u8")
MJPEG("MJPEG 帧流", PlayKind.MJPEG, "mjpeg")
// PlayKind: NATIVE / HLS / MJPEG（前端三态播放壳）

// RunResult（record）
String runId; TranscodeMode mode; boolean streaming; boolean success; int exitCode;
String command; Long firstByteMs; Long totalMs; long outputBytes;
List<String> stderrTail; long timestamp;

// FfmpegLabProperties（toolbox.ffmpeg-lab.*）
String workDir = "${user.home}/.kai-toolbox/ffmpeg-lab";
int defaultClipSeconds = 30;
int retainMinutes = 30;        // workDir 子目录保留期
int stderrTailLines = 40;
```

### 各模式命令骨架（ModeCommandBuilder.build）

```
公共前缀: <binary> -hide_banner -loglevel warning -y [-t <clip> 当 clip>0]
VENC = -vf scale=w=max(iw\,256):h=max(ih\,144):force_original_aspect_ratio=increase:force_divisible_by=2 -pix_fmt yuv420p -c:v libx264 -preset veryfast -crf 23
AENC = -c:a aac -ar 44100 -b:a 128k
REMUX_COPY      : -i <in> -c copy -movflags +faststart -f mp4 <workDir>/out.mp4
PROGRESSIVE_MP4 : -i <in> <VENC> <AENC> -movflags +faststart -f mp4 <workDir>/out.mp4
HLS_TS          : -i <in> <copy|VENC> <copy|AENC> -f hls -hls_time 10 -hls_segment_type mpegts -hls_playlist_type vod -hls_flags independent_segments <workDir>/index.m3u8
HLS_FMP4        : -i <in> <VENC> <AENC> -f hls -hls_time 10 -hls_segment_type fmp4 -hls_playlist_type vod <workDir>/index.m3u8
MJPEG           : -i <in> -an -f mpjpeg -q:v 5 pipe:1
```
> **一律软编 libx264**（不继承 hwaccel，避开 NVENC 对超小帧的最小尺寸限制）。VENC 的 scale 把小帧放大到 ≥256x144 保宽高比偶数尺寸；AENC 重采样 44100。HLS copy 判定复用 FfmpegProbe.canCopyVideo/Audio；clip 用 `-t`（放 -i 后，简单稳）。

---

## 5. 重要约束与边界

- 路径解析：`Path.of(path).normalize()`，校验 `Files.isRegularFile`；本地单用户工具不做 scan 域限制（同 media-parser）。
- play 端点：`REMUX_COPY/PROGRESSIVE_MP4` 用 `ResourceRegion` 支持 Range（仿 RawStreamService），HLS 返回 m3u8 文本 + 分段静态托管同 runId 前缀，MJPEG 用 `StreamingResponseBody` + content-type `multipart/x-mixed-replace; boundary=ffmpeg`。
- 并发：无锁，每 run 独立 workDir；同一文件多模式互不干扰。
- 不处理：DASH/WebM（本期排除）；不落库；不做生产级按需分段流式。

---

## 6. 下游依赖调用

```
com.exceptioncoder.toolbox.common.media.FfmpegProbe#probe(Path): ProbeResult
FfmpegProbe#isFfmpegAvailable(): boolean / #canCopyVideo / #canCopyAudio / #nativelyPlayable
FfmpegProcessRegistry#spawn(ProcessBuilder): Process / #activeCount(): int
FfmpegProperties#getBinary / #getHwaccel
```

---

## 7. 异常处理要点

- ffmpeg 不可用 → 抛 `com.exceptioncoder.toolbox.common.media.FfmpegUnavailableException`（全局→503）。
- 文件不存在 → `NoSuchFileException` 或 404 ResponseEntity。
- ffmpeg 退出码≠0 → 不抛异常，`RunResult.success=false` + `stderrTail`，HTTP 200 返回。
- 客户端断开（StreamingResponseBody 写失败）→ 强杀进程，debug 日志，record success=false（aborted）。
- workDir 清理失败 → 仅 warn，不阻断运行。

---

## 8. 前端要点

- `index.tsx`：FeatureManifest，icon 取 lucide（如 `FlaskConical`/`Clapperboard`），group `视频工具`（与 video-library 同组）或 `开发工具`，route `/tools/ffmpeg-lab`。
- `api.ts`：`probe(path,clip)` / `run(body)` / `playUrl(mode,runId|path,clip)` / `recentRuns()`，复用 `@/lib/api` 的 `http`。
- `LabPlayer.tsx`：按 playKind → `native`=`<video src controls>`；`hls`=hls.js 加载 playUrl（Safari 原生回退）；`mjpeg`=`<img src>`。卸载时 `hls.destroy()` + `video.pause()`（仿 video-playback/VideoPlayer）。
- `ModeCard.tsx`：显示 label、预判 badge（OK 绿 / TRANSCODE 蓝 / FAIL 红）、命令折叠区、运行按钮（主 CTA 用 lg+shadow，遵循团队按钮强调规范）。
- `RunDiagnosticsTable.tsx`：useQuery 轮询 `/runs/recent`（2s），列 mode/success/exitCode/耗时/产物大小/stderr 尾部。
- 注释中文（kai-toolbox 规范）。
