# 语音与附件输入 — 编码摘要

> 配套 [语音与附件输入-current.md](语音与附件输入-current.md) 与 [语音与附件输入-api-current.md](语音与附件输入-api-current.md)。
> 本文件是编码前的方法级落点摘要；字段级接口契约以 api 文档为准，不在此重复。
> 最后更新日期：2026-06-01

## 0. 本期已拍板默认

- 附件落盘：会话 cwd 内 `.kai-chat-attachments/{sessionId}/`，会话删除即清理。
- ASR 复用：`WhisperAsrClient` 下沉 `toolbox-common`，`tool-treesize` 改依赖共享实现。
- 上限：单文件 ≤ 50MB，单条消息 ≤ 10 个附件。

## 1. 核心业务规则

- 语音只转文字，回填可编辑再发；原始音频转写后删除，不持久化。
- 附件必须落 cwd 内，Claude 才能 Read；路径引用而非 base64 内联。
- `send` 不带 attachments 时行为完全不变（向前兼容）。
- sidecar 协议与 `SidecarClient.userMessage` 签名不变——附件路径在 Java 侧拼进 text。
- faster-whisper 不在线 → STT 返回 `ASR_UNAVAILABLE`，前端禁用麦克风。

## 2. 接口入口指针（契约见 api 文档）

| 方法 + 路径 | 实现类#方法 |
|---|---|
| `POST /api/claude-chat/stt` | `SttController#transcribe` |
| `POST /api/claude-chat/sessions/{id}/attachments` | `AttachmentController#upload` |
| WS `send {text, attachments?}` | `ClaudeChatService#sendUserMessage`（改） |

## 3. 涉及类清单（全路径 + 方法签名 + 职责）

### 后端 · toolbox-common（共享能力下沉）

- `com.exceptioncoder.toolbox.common.speech.SpeechToTextClient` [新增]
  - `String transcribeToText(Path wav, String language)` — 调 faster-whisper `/asr`，解析 SSE，把 segment 合并为纯文本（去时间轴/VTT 标签）返回。
  - `Path transcribeToVtt(Path wav, Path outputPrefix, String language, String initialPrompt, ProgressListener l, AtomicBoolean cancelled)` — 保留原 VTT 能力，供 treesize 字幕复用（即原 `WhisperAsrClient#run` 迁移）。
  - `boolean isAvailable()` / `String pingHealth()` — 探活。
- `com.exceptioncoder.toolbox.common.speech.WhisperProperties` [迁移] — 由 treesize 下沉；`@ConfigurationProperties("toolbox.whisper")` 保持不变。
- `tool-treesize` 的 `WhisperAsrClient` / `SubtitleService` [改] — 改为依赖 common 的 `SpeechToTextClient`，删除重复实现；签名对齐，调用点最小改动。

### 后端 · tool-claude-chat

- `…claudechat.api.SttController` [新增]
  - `ResponseEntity<SttResult> transcribe(InputStream body, @RequestParam(defaultValue="auto") String language, @RequestHeader HttpHeaders h)` — raw body 收音频，委托 `SttService`。
- `…claudechat.api.AttachmentController` [新增]
  - `AttachmentView upload(@PathVariable String sessionId, @RequestPart("file") MultipartFile file)`。
- `…claudechat.api.dto.SttResult` [新增] — `record SttResult(String text)`。
- `…claudechat.api.dto.AttachmentView` [新增] — `record AttachmentView(String id, String name, String mime, long size, String path)`。
- `…claudechat.service.SttService` [新增]
  - `String transcribe(InputStream audio, String contentType, String language)` — ① 写临时文件 → ② ffmpeg 转 16kHz mono wav（复用 `AudioExtractor` 范式）→ ③ `SpeechToTextClient.transcribeToText` → ④ 清理临时文件 → 返回文本。
- `…claudechat.service.AttachmentStorageService` [新增]
  - `AttachmentView store(String sessionId, MultipartFile file)` — 解析会话 cwd（查 `ClaudeChatSessionRepository`）→ 校验大小/类型 → 文件名消毒 → 落盘 `{cwd}/.kai-chat-attachments/{sid}/{ts}-{name}` → 返回句柄。
  - `void clear(String sessionId)` — 删除该会话附件目录（接到 `dropSession`）。
- `…claudechat.api.dto.ClientMessage` [改] — `Send` 记录加 `List<Attachment> attachments`，内嵌 `record Attachment(String name, String path)`。
- `…claudechat.service.ClaudeChatService#sendUserMessage` [改] — 若 `attachments` 非空，调私有 `appendAttachmentHints(text, attachments)` 拼结构化提示，再 `sidecar.userMessage(sessionId, finalText)`；`dropSession` 调 `AttachmentStorageService.clear`。

### 前端 · features/claude-chat

- `hooks/useVoiceRecorder.ts` [新增] — `{ recording, start(), stop():Promise<Blob>, cancel(), seconds }`，封装 `getUserMedia`+`MediaRecorder`。
- `components/VoiceInputButton.tsx` [新增] — 录音按钮，停止后 `transcribe(blob)` → 回调 `onText(text)`；ASR 不可用时禁用并提示。
- `components/AttachmentChips.tsx` [新增] — 渲染附件 chip（图片缩略图 + 删除）。
- `pages/ChatPage.tsx` [改] — 输入区加 🎤/📎 + chip 区；维护 `attachments` state；`submit` 调 `chat.send(draft, attachments)` 后清空。
- `hooks/useClaudeChatSocket.ts` [改] — `send(text: string, attachments?: Attachment[])`，user 气泡仍只显示 text，`sendRaw({type:'send', text, attachments})`。
- `api.ts` [改] — `transcribe(blob, language?)` → `POST /stt`；`uploadAttachment(sessionId, file)` → `POST .../attachments`。
- `types.ts` [改] — `ClientMessage` 的 send 加 `attachments?`；新增 `interface Attachment { name; path }`。

## 4. 数据结构

- 无新增数据库表/字段。附件为临时文件，元数据仅在内存/响应里流转。
- 落盘目录：`{cwd}/.kai-chat-attachments/{sessionId}/{epochMillis}-{sanitizedName}`。

## 5. 重要约束与边界

- ffmpeg/whisper 调用走虚拟线程，不阻塞 WS 线程。
- 临时音频文件 try-finally 必删；STT 失败也要清理。
- 文件名消毒：去路径分隔符与 `..`，限长；冲突加序号。
- 类型白名单：image/*、pdf、纯文本/常见文档；拒可执行。
- 注释中文（kai-toolbox 约定），仅符号保留英文。

## 6. 测试 / 验证要点

- STT：中文录音转写回填正确；服务停 → `ASR_UNAVAILABLE` 且麦克风禁用；空音频不报错。
- 附件：图+PDF 上传 → chip → 发送 → Claude 触发 Read 命中文件。
- 兼容：`send` 不带 attachments 时旧路径零变化。
- 回归：treesize 字幕在 ASR 下沉后仍出 VTT；`mvn package` fat-jar 正常。
