# 在线视频播放 — 接口契约

> 最后更新：2026-05-05
> 配套：`在线视频播放-current.md`

## 接口清单

| Method | Path | 说明 | 实现类#方法 |
|--------|------|------|------------|
| `GET` | `/api/treesize/config` | 拉视频白名单 + ffmpeg 可用性 | `TreeSizeController#videoConfig` |
| `HEAD` | `/api/treesize/scans/{id}/probe` | 探测文件是否能原生直放 | `TreeSizeController#probeFile` |
| `GET` | `/api/treesize/scans/{id}/stream` | 直放（带 Range） | `TreeSizeController#streamRaw` |
| `GET` | `/api/treesize/scans/{id}/hls/playlist.m3u8` | HLS 播放列表（动态生成） | `TreeSizeController#hlsPlaylist` |
| `GET` | `/api/treesize/scans/{id}/hls/segment-{idx}.ts` | HLS 分片（按需转码到 stdout） | `TreeSizeController#hlsSegment` |

`/probe` `/stream` `/hls/*` 都强制 `path` 必须落在该 scan 的源根目录下，否则 400。

---

## 1. GET /api/treesize/config

**用途**：前端启动期拉一次，拿到视频扩展名白名单和 ffmpeg 可用状态。

### 请求

无参数。

### 响应（200）

```json
{
  "videoExtensions": ["mp4", "m4v", "mov", "webm", "ogv", "mkv", "avi", "flv", "wmv", "rmvb", "3gp", "ts", "mts", "m2ts", "vob", "asf", "divx", "mpeg", "mpg", "m2v"],
  "ffmpegAvailable": true
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `videoExtensions` | `string[]` | 不带前导点的扩展名（小写） |
| `ffmpegAvailable` | `boolean` | 启动期探测结果，影响转码可用性 |

---

## 2. HEAD /api/treesize/scans/{id}/probe

**用途**：决定播放器走 `/stream` 还是 `/hls/playlist.m3u8`。HEAD 方法，结果全在 header。

### 请求

| 参数 | 位置 | 必填 | 说明 |
|------|------|-----|------|
| `id` | path | 是 | scan id |
| `path` | query | 是 | 文件绝对路径（URL encoded） |

### 响应（200）

| Header | 取值 | 说明 |
|--------|------|------|
| `X-Native-Playable` | `true` / `false` | 浏览器是否能直放 |
| `X-Container` | `mp4` / `mkv` / `flv` / … | ffprobe 识别的容器；ffprobe 不可用时返回 `unknown` |
| `X-Video-Codec` | `h264` / `hevc` / `av1` / … | 同上 |
| `X-Audio-Codec` | `aac` / `ac3` / `(none)` / … | 无音频流时 `(none)` |
| `X-Duration-Seconds` | `1234.567` | ffprobe 拿到的时长，HLS 分片数由它决定 |
| `X-Ffmpeg-Available` | `true` / `false` | 后端是否检测到 ffmpeg；为 false 时不走 HLS |

### 错误

| 状态 | 触发条件 | body |
|------|---------|------|
| 400 | path 越权 / 不是普通文件 | `{message: "..."}` |
| 404 | 文件不存在 | `{message: "file not found"}` |

---

## 3. GET /api/treesize/scans/{id}/stream

**用途**：原生格式字节级直放，支持 HTTP Range。

### 请求

| 参数 | 位置 | 必填 | 说明 |
|------|------|-----|------|
| `id` | path | 是 | scan id |
| `path` | query | 是 | 文件绝对路径 |
| `Range` | header | 否 | 标准 HTTP Range，例：`bytes=1024-` |

### 响应

| 状态 | 场景 | Headers |
|------|------|---------|
| 200 | 无 Range | `Content-Type`、`Content-Length`、`Accept-Ranges: bytes` |
| 206 | 有 Range | `Content-Type`、`Content-Range: bytes a-b/total`、`Content-Length`、`Accept-Ranges: bytes` |
| 416 | Range 越界 | `Content-Range: bytes */total` |

`Content-Type` 按扩展名映射：`.mp4/.m4v/.mov → video/mp4`，`.webm → video/webm`，`.ogv → video/ogg`，未知 → `application/octet-stream`。

错误响应同 `/probe`。

---

## 4. GET /api/treesize/scans/{id}/hls/playlist.m3u8

**用途**：返回动态生成的 HLS 播放列表，hls.js 喂给 `<video>` 元素。

### 请求

| 参数 | 位置 | 必填 | 说明 |
|------|------|-----|------|
| `id` | path | 是 | scan id |
| `path` | query | 是 | 文件绝对路径 |

### 响应

成功（200）：

```
Content-Type: application/vnd.apple.mpegurl

#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-PLAYLIST-TYPE:VOD
#EXTINF:6.0,
segment-0.ts?path=<encoded>
#EXTINF:6.0,
segment-1.ts?path=<encoded>
...
#EXTINF:3.5,
segment-N.ts?path=<encoded>
#EXT-X-ENDLIST
```

| 错误状态 | 触发条件 | body |
|---------|---------|------|
| 503 | ffmpeg / ffprobe 不可用 | `{message: "FFmpeg 不可用，请在 application.yml 配置 toolbox.ffmpeg.binary"}` |
| 400 | path 越权 / 不是普通文件 | `{message: "..."}` |
| 404 | 文件不存在 | `{message: "file not found"}` |

> 实现注意：m3u8 中分片 URI 必须带上 `?path=...`，因为这是相对 URI，hls.js 会按当前 m3u8 路径解析。或者直接使用绝对路径如 `/api/treesize/scans/{id}/hls/segment-{idx}.ts?path=...`。

---

## 5. GET /api/treesize/scans/{id}/hls/segment-{idx}.ts

**用途**：按需转码并流出第 idx 个分片。

### 请求

| 参数 | 位置 | 必填 | 说明 |
|------|------|-----|------|
| `id` | path | 是 | scan id |
| `idx` | path | 是 | 分片序号，0-based |
| `path` | query | 是 | 文件绝对路径 |

### 响应

| 状态 | 场景 | Headers / Body |
|------|------|----------------|
| 200 | 正常 | `Content-Type: video/mp2t`，**不带** `Content-Length`（chunked transfer），body 是 mpegts 字节流 |
| 503 | ffmpeg 不可用 | JSON `{message: "..."}` |
| 400 | path 越权 / idx 越界（idx >= 总片数） | JSON |
| 404 | 文件不存在 | JSON |
| 500 | ffmpeg 启动失败 / 进程异常退出 | JSON `{message: "transcode failed: <stderr 末尾几行>"}` |

### 客户端断开行为

`<video>` 卸载 / hls.js abort → 浏览器关 TCP → 后端 response IOException → HlsService **必须** `process.destroyForcibly()`，stderr drain 线程必须退出。

---

## 错误格式（统一）

```json
{
  "timestamp": "2026-05-05T20:11:35.240Z",
  "status": 400,
  "error": "Bad Request",
  "message": "path outside scan root"
}
```

由现有 `GlobalExceptionHandler` 输出。
