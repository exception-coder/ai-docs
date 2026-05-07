# 在线视频播放 — 编码摘要

> 配套：`在线视频播放-current.md`、`在线视频播放-api-current.md`
> 仅用于编码阶段快速定位入口、方法签名、关键约束。

## 1. 核心业务规则（编码必须遵守）

- **直放/转码二选一**：仅当容器 ∈ {mp4, m4v, webm, ogg, mov} 且 视频编码 ∈ {h264, vp8, vp9, av1} 且（无音频流 或 音频编码 ∈ {aac, mp3, opus, vorbis}）→ 走 `/stream`；否则走 `/hls/*`。
- **分片长度固定 6 秒**，最后一片可能更短，由 `Math.min(6, duration - idx*6)` 决定。
- **fast-path**：原始视频编码已是 h264 且音频已是 aac/mp3，HLS 分片用 `-c:v copy -c:a copy`；其它 `-c:v libx264 -preset veryfast -c:a aac`。
- **路径越权**：所有按 path 读盘的接口必须先过 `PathAccessGuard`。
- **进程清理**：FFmpeg 进程必须随 HTTP 请求生命周期清理，client 断开 → `destroyForcibly()`。
- **零落盘**：playlist 内存拼接，segment stdout 管道直传，绝对禁止落盘。

## 2. 接口入口指针

| 端点 | 实现方法 | 备注 |
|------|---------|------|
| GET /api/treesize/config | `TreeSizeController#videoConfig` | 字段级契约见 api-doc 第 1 节 |
| HEAD /api/treesize/scans/{id}/probe | `TreeSizeController#probeFile` | api-doc 第 2 节 |
| GET /api/treesize/scans/{id}/stream | `TreeSizeController#streamRaw` | api-doc 第 3 节 |
| GET /api/treesize/scans/{id}/hls/playlist.m3u8 | `TreeSizeController#hlsPlaylist` | api-doc 第 4 节 |
| GET /api/treesize/scans/{id}/hls/segment-{idx}.ts | `TreeSizeController#hlsSegment` | api-doc 第 5 节 |

## 3. 涉及类清单

### 后端（toolbox-common 新增）

```java
// 包: com.exceptioncoder.toolbox.common.media

@ConfigurationProperties(prefix = "toolbox.ffmpeg")
public class FfmpegProperties {
    private String binary = "ffmpeg";
    private String ffprobeBinary = "ffprobe";
    private long probeTimeoutMs = 5000;
    // getters / setters
}

public record ProbeResult(double durationSeconds, String container, String videoCodec, String audioCodec) {
    public static final ProbeResult UNKNOWN = new ProbeResult(0, "unknown", "unknown", "(none)");
}

@Component
public class FfmpegProbe {
    public FfmpegProbe(FfmpegProperties props);

    @PostConstruct
    public void detect();                                           // 启动期跑 `<binary> -version`，缓存 available

    public boolean isFfmpegAvailable();                              // 启动探测结果

    public ProbeResult probe(Path file) throws IOException;          // ffprobe 拿元信息，按 (path, mtime) LRU 缓存

    public boolean nativelyPlayable(ProbeResult r);                  // 第 5 节规则
}
```

### 后端（tool-treesize 新增）

```java
// 包: com.exceptioncoder.toolbox.treesize.config

@ConfigurationProperties(prefix = "toolbox.video")
public class VideoExtensionsProperties {
    private List<String> extensions = List.of();                     // 默认值在 application.yml
    // getters / setters
}

// 包: com.exceptioncoder.toolbox.treesize.api.dto

public record VideoConfigView(List<String> videoExtensions, boolean ffmpegAvailable) {}

// 包: com.exceptioncoder.toolbox.treesize.service

@Component
public class PathAccessGuard {
    public PathAccessGuard(ScanRepository scans);
    /**
     * 取出 scan rootPath，toRealPath；请求 path toRealPath；要求后者 startsWith 前者。
     * 不通过抛 IllegalArgumentException("path outside scan root")。
     * 文件不存在抛 NoSuchFileException → controller 翻 404。
     */
    public Path resolve(String scanId, String requestedPath) throws IOException;
}

@Component
public class RawStreamService {
    /**
     * 处理 Range 头返回 ResourceRegion 或全文件。
     * Content-Type 按扩展名映射；未知 → application/octet-stream。
     */
    public ResponseEntity<Resource> serve(Path file, HttpHeaders requestHeaders) throws IOException;
}

@Component
public class HlsService {
    private static final int SEGMENT_SECONDS = 6;

    public HlsService(FfmpegProbe probe, FfmpegProperties props);

    /** 内存拼 m3u8 字符串。 */
    public String playlist(String scanId, Path file) throws IOException;

    /**
     * fork ffmpeg -ss <idx*6> -t 6 -i path -c:v ... -f mpegts pipe:1。
     * stdout 拷到 out，stderr 单线程 drain 到 DEBUG 日志。
     * client 断开（IOException on out.write）→ destroyForcibly。
     * idx 越界 → IndexOutOfBoundsException → controller 翻 404。
     */
    public void writeSegment(Path file, int idx, OutputStream out) throws IOException;
}
```

### 后端（tool-treesize 修改）

```java
// TreeSizeController 新增方法（去 SseEmitter 之外的现有方法保持不变）

@GetMapping("/config")
public VideoConfigView videoConfig();

@RequestMapping(value = "/scans/{id}/probe", method = RequestMethod.HEAD)
public ResponseEntity<Void> probeFile(@PathVariable String id, @RequestParam String path);

@GetMapping("/scans/{id}/stream")
public ResponseEntity<Resource> streamRaw(@PathVariable String id,
                                           @RequestParam String path,
                                           @RequestHeader HttpHeaders headers);

@GetMapping(value = "/scans/{id}/hls/playlist.m3u8",
            produces = "application/vnd.apple.mpegurl")
public ResponseEntity<String> hlsPlaylist(@PathVariable String id, @RequestParam String path);

@GetMapping(value = "/scans/{id}/hls/segment-{idx}.ts",
            produces = "video/mp2t")
public StreamingResponseBody hlsSegment(@PathVariable String id,
                                         @PathVariable int idx,
                                         @RequestParam String path);
```

控制器只做参数解析、`PathAccessGuard.resolve` 鉴权，然后委派给 service。

### 前端（features/treesize 新增/修改）

```ts
// types.ts 新增
export interface VideoConfig {
  videoExtensions: string[]
  ffmpegAvailable: boolean
}

export interface ProbeResult {
  nativelyPlayable: boolean
  container: string
  videoCodec: string
  audioCodec: string
  durationSeconds: number
  ffmpegAvailable: boolean
}

// api.ts 新增
export function getVideoConfig(): Promise<VideoConfig>
export function probeVideo(scanId: string, path: string): Promise<ProbeResult>
export function streamUrl(scanId: string, path: string): string
export function hlsPlaylistUrl(scanId: string, path: string): string

// utils.ts 新增
export function isVideoFile(name: string, whitelist: string[]): boolean

// hooks/useVideoConfig.ts 新增
export function useVideoConfig(): VideoConfig | undefined

// components/VideoPlayerModal.tsx 新增（默认 export 一个组件）
interface VideoPlayerModalProps {
  scanId: string
  path: string
  name: string
  onClose: () => void
}
// 内部行为：
//   1. probeVideo() 拿 ProbeResult
//   2. 如果 nativelyPlayable → <video src={streamUrl(...)}>
//   3. 否则如果 ffmpegAvailable → 用 hls.js loadSource(hlsPlaylistUrl(...))
//   4. 否则 → 显示"格式不支持，且 FFmpeg 不可用"提示

// components/ChildrenList.tsx 修改
//   - 接收 videoExtensions prop
//   - 文件 onClick：isVideoFile(n.name, videoExtensions) → 打开 VideoPlayerModal；否则保持现有行为
```

## 4. 数据结构

无 DB 变更。配置项：

```yaml
toolbox:
  ffmpeg:
    binary: ffmpeg
    ffprobe-binary: ffprobe
    probe-timeout-ms: 5000
  video:
    extensions: [mp4, m4v, mov, webm, ogv, mkv, avi, flv, wmv, rmvb, 3gp, ts, mts, m2ts, vob, asf, divx, mpeg, mpg, m2v]
```

## 5. 重要约束与边界

- **进程生命周期**：HlsService 内部用 try-with-resources 把 `Process` 包起来（实现 `AutoCloseable` 包装 / 或 finally 块），确保任何路径退出都 `destroyForcibly`，且 stderr drain 线程 join 退出。
- **stderr drain**：必须用单独线程读 `process.getErrorStream()`，否则 stderr buffer 满会让 ffmpeg 挂起。
- **路径校验**：先 `Path.of(rootPath).toRealPath()`，再 `Path.of(requestPath).toRealPath()`，比较后者是否 startsWith 前者。`toRealPath` 会解析 symlink。如果 requestPath 不存在 → `NoSuchFileException` → 404。
- **Range 解析**：用 `HttpHeaders.getRange()`（Spring 自带），不手撸 parser。
- **m3u8 分片 URL**：用相对路径 `segment-{idx}.ts?path=<encoded>`，让 hls.js 按 m3u8 自身路径解析；这样不依赖部署前缀。
- **fast-path 决策**：在 `HlsService.writeSegment` 内部基于 `FfmpegProbe.probe(file)` 的 codec 信息选 `-c copy` 还是重新编码。codec 兼容判定：videoCodec=="h264" && audioCodec ∈ {"aac","mp3","(none)"}。
- **probe LRU 缓存**：用 `Collections.synchronizedMap(new LinkedHashMap<...>(1024, 0.75f, true) { override removeEldestEntry })` 即可，不引入 Caffeine。
- **错误码翻译**：Controller 把 service 抛的异常翻译成 HTTP 状态：`NoSuchFileException` → 404；`IllegalArgumentException` → 400（已有 GlobalExceptionHandler 处理）；`IllegalStateException("ffmpeg unavailable")` → 503，需在 GlobalExceptionHandler 加一个 handler。
- **bugfix-coding-style**：禁止在源码内写变更日志注释、日期、PR 编号；类/方法 doc 只写当前职责；复杂代码块上方加短 WHY 注释。
- **Java 版本**：Java 21，可用 `record`、`var`、虚拟线程（与现有代码一致）。

## 6. 测试要点（手测，不写自动化测试）

- mp4 直放：拖进度条 OK
- mkv（HEVC + AC3）：转码后能播；拖进度条 OK
- flv：转码后能播
- 关闭 modal 后 `Get-Process ffmpeg` 应没有遗留进程（Windows）
- application.yml 把 `binary` 改成不存在的路径：前端 `/config` 应该 `ffmpegAvailable=false`，转码场景显示错误而非崩溃
- 点击非视频文件（白名单外）：保持原行为（现在是无操作）
