# FFmpeg 转码实验台 接口文档

> 最后更新日期：2026-05-31
> 配套技术方案：[FFmpeg转码实验台-current.md](FFmpeg转码实验台-current.md)
> 字段级契约在本文档维护；设计决策/流程/风险见技术方案文档。

---

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|--------------|
| v1 | 2026-05-31 | ping.yang | 初始版本 |

---

## 接口清单

| # | 方法 | 路径 | 用途 |
|---|------|------|------|
| 1 | GET | /api/ffmpeg-lab/probe | 探测文件 + 每模式预判 + 命令预览 |
| 2 | POST | /api/ffmpeg-lab/run | 实跑某模式（临时文件类阻塞返回诊断）|
| 3a | GET | /api/ffmpeg-lab/play/file/{runId}/{name} | 托管 mp4 产物（Range，progressive/remux）|
| 3b | GET | /api/ffmpeg-lab/play/hls/{runId}/{name} | 托管 HLS 物料（m3u8 + ts/m4s/init）|
| 3c | GET | /api/ffmpeg-lab/play/mjpeg | MJPEG multipart 流式 |
| 4 | GET | /api/ffmpeg-lab/runs/recent | 最近运行诊断（前端轮询）|

> 播放端点按投递类型拆为三个，由 /run 返回的 `playUrl` 直接给前端播放壳；前端不手工拼这些地址。

> 认证：全部不需要（本地单用户工具）。

---

## 1. 探测 + 模式预判

### 1.1 基本信息

- **方法**：`GET`
- **路径**：`/api/ffmpeg-lab/probe`
- **用途**：ffprobe 探测容器/编码/时长，并对 5 种模式给出预判与将执行的命令。
- **幂等**：是。

### 1.2 请求

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `path` | string(query) | 是 | - | 本地绝对路径，URL 编码 |
| `clipSeconds` | integer(query) | 否 | 30 | 命令预览用的截断秒数，0=整片 |

### 1.3 响应

#### 成功（200）

```json
{
  "ffmpegAvailable": true,
  "probe": {
    "container": "mov,mp4,m4a,3gp,3g2,mj2",
    "videoCodec": "mpeg4",
    "audioCodec": "qcelp",
    "durationSeconds": 10.13,
    "nativelyPlayable": false
  },
  "modes": [
    {
      "mode": "REMUX_COPY",
      "label": "Remux 直封装",
      "playKind": "native",
      "prediction": "FAIL",
      "predictionReason": "视频 mpeg4 / 音频 qcelp 非 mp4 原生兼容，copy 无法出 web",
      "command": "ffmpeg -i <in> -t 30 -c copy -movflags +faststart -f mp4 <out.mp4>"
    },
    {
      "mode": "PROGRESSIVE_MP4",
      "label": "Progressive MP4 全转码",
      "playKind": "native",
      "prediction": "TRANSCODE",
      "predictionReason": "重编码到 H.264/AAC，通用兜底",
      "command": "ffmpeg -i <in> -t 30 -c:v libx264 -preset veryfast -crf 23 -c:a aac -b:a 128k -movflags +faststart -f mp4 <out.mp4>"
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `ffmpegAvailable` | boolean | ffmpeg 是否可用，false 时运行端点将返回 503 |
| `probe` | object | ffprobe 结果；探测失败时各字段为 unknown |
| `probe.nativelyPlayable` | boolean | 浏览器是否可直接原生播放原文件 |
| `modes[].mode` | string | 模式枚举：REMUX_COPY / PROGRESSIVE_MP4 / HLS_TS / HLS_FMP4 / MJPEG |
| `modes[].playKind` | string | 投递类型：`native` / `hls` / `mjpeg` |
| `modes[].prediction` | string | `OK`（可原生/copy）/ `TRANSCODE`（需转码，预期可成）/ `FAIL`（预判不可行）|
| `modes[].command` | string | 将执行的 ffmpeg 命令（与 /run 实跑逐字一致）|

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `INVALID_PARAM` | 400 | path 为空 |
| `FILE_NOT_FOUND` | 404 | 文件不存在 |

---

## 2. 运行某模式

### 2.1 基本信息

- **方法**：`POST`
- **路径**：`/api/ffmpeg-lab/run`
- **用途**：实跑指定模式。临时文件类模式（REMUX_COPY/PROGRESSIVE_MP4/HLS_TS/HLS_FMP4）阻塞至 ffmpeg 退出，返回诊断 + 播放地址；流式模式（MJPEG）直接返回 playUrl，诊断在流结束后经接口 4 回填。
- **幂等**：否（每次产生新 runId）。

### 2.2 请求

#### Body

```json
{
  "path": "C:\\Users\\张凯\\Desktop\\tmp\\360653.amc",
  "mode": "PROGRESSIVE_MP4",
  "clipSeconds": 30
}
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `path` | string | 是 | - | 本地绝对路径 |
| `mode` | string | 是 | - | 模式枚举 |
| `clipSeconds` | integer | 否 | 30 | 0=整片 |

### 2.3 响应

#### 成功（200）

```json
{
  "runId": "a1b2c3",
  "mode": "PROGRESSIVE_MP4",
  "streaming": false,
  "success": true,
  "exitCode": 0,
  "command": "ffmpeg -i ... -f mp4 out.mp4",
  "firstByteMs": null,
  "totalMs": 1840,
  "outputBytes": 524288,
  "stderrTail": ["frame= 152 fps=...", "..."],
  "playUrl": "/api/ffmpeg-lab/play/PROGRESSIVE_MP4?runId=a1b2c3",
  "playKind": "native"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `runId` | string | 本次运行 id，play 端点用它定位物料 |
| `streaming` | boolean | true=流式模式，诊断字段稍后经接口 4 回填 |
| `success` | boolean | 是否能真正输出到 web：NATIVE 类要求 ffmpeg exit0 + 产物存在 + 产物编码浏览器原生可播（封装成功但编码不兼容仍为 false）；HLS 类 exit0+playlist；流式看字节数 |
| `firstByteMs` | integer/null | 流式模式首字节耗时；临时文件类为 null |
| `totalMs` | integer | 临时文件类总转码耗时 |
| `outputBytes` | integer | 产物字节数（流式为流结束后累计）|
| `stderrTail` | string[] | ffmpeg stderr 尾部 N 行 |
| `playKind` | string | `native` / `hls` / `mjpeg` |

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| `FILE_NOT_FOUND` | 404 | 文件不存在 |
| `FFMPEG_UNAVAILABLE` | 503 | ffmpeg 不可用 |
| `INVALID_PARAM` | 400 | mode 非法 |

> 注：ffmpeg 退出码非 0（如 Remux 编码不兼容）仍返回 **200**，`success=false` + `stderrTail` 给出原因——失败本身是实验台要呈现的结果，不当作 HTTP 错误。

---

## 3. 播放 / 流式端点

### 3a. 托管 mp4 产物（progressive / remux）

- **方法**：`GET` ｜ **路径**：`/api/ffmpeg-lab/play/file/{runId}/{name}`（`name` 当前恒为 `out.mp4`）
- **Content-Type**：`video/mp4`，支持 Range（无 Range→200 全量，有 Range→206 + `Content-Range`，均带 `Accept-Ranges: bytes`）。
- 路径穿越防护：`{name}` 解析后必须落在该 runId 目录内且为常规文件，否则 404。

### 3b. 托管 HLS 物料（hls-ts / hls-fmp4）

- **方法**：`GET` ｜ **路径**：`/api/ffmpeg-lab/play/hls/{runId}/{name}`
- playlist 入口 `name=index.m3u8`；分段相对名（`index0.ts` / `index0.m4s` / `init.mp4`）由浏览器对 m3u8 URL 解析后请求同前缀。
- **Content-Type**：`.m3u8`→`application/vnd.apple.mpegurl`、`.ts`→`video/mp2t`、`.m4s`/`.mp4`→`video/mp4`。

### 3c. MJPEG 帧流

- **方法**：`GET` ｜ **路径**：`/api/ffmpeg-lab/play/mjpeg`
- **Query**：`path`（源文件，必填）、`runId`（必填，诊断回填用）、`clipSeconds`（可选）。
- **Content-Type**：`multipart/x-mixed-replace; boundary=ffmpeg`，连续 JPEG 帧，无音频。流结束（完成/断开/失败）后诊断经接口 4 回填。

#### 错误

| 错误码 | HTTP 状态 | 触发场景 |
|--------|-----------|---------|
| 物料不存在 | 404 | runId/name 越界、不存在或已被清理（NoSuchFileException）|

---

## 4. 最近运行诊断

### 4.1 基本信息

- **方法**：`GET`
- **路径**：`/api/ffmpeg-lab/runs/recent`
- **用途**：前端轮询刷新诊断表，含流式模式回填后的结果。

### 4.2 响应

#### 成功（200）

```json
{
  "activeFfmpegCount": 1,
  "runs": [
    {
      "runId": "a1b2c3",
      "mode": "PROGRESSIVE_MP4",
      "success": true,
      "exitCode": 0,
      "firstByteMs": null,
      "totalMs": 1840,
      "outputBytes": 524288,
      "stderrTail": ["..."],
      "timestamp": 1748678400000
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `activeFfmpegCount` | integer | 当前活跃 ffmpeg/ffprobe 进程数（透传 FfmpegProcessRegistry）|
| `runs` | object[] | 最近 ~50 条运行诊断，按时间倒序 |
