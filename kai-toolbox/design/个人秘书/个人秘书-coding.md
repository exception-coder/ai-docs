# 个人秘书 · 编码摘要

> **关联设计文档**：`个人秘书-current.md`（完整-技术）
> **最后更新**：2026-05-18
> **范围**：kai-toolbox / frontend 新增 feature 模块 `secretary`

---

## 1. 核心业务规则

1. 每条 entry 在工厂层注入 `id (crypto.randomUUID)` + `createdAt (Date.now())` + `inputMethod`，调用方不允许传入。
2. `geo` 字段尽力而为：4 秒超时 / 用户拒绝 / 浏览器不支持 → `null`，不阻塞、不弹错。
3. 文字消息提交前 `text.trim().length > 0`，纯空白拒绝。
4. 附件多选时一文件一条 entry，便于时间轴。
5. 语音 mime 优先 `audio/webm;codecs=opus`，不支持时退化默认。
6. 删除 entry 时同步删除 `blobs` store 中同 id 的二进制；无回收站。

---

## 2. 入口指针（路由 → 实现）

| 路由 | 实现入口 |
|------|---------|
| `/tools/secretary` | `frontend/src/features/secretary/pages/SecretaryPage.tsx` `SecretaryPage` |

侧边栏注册：`frontend/src/features/secretary/index.tsx` 默认导出 `FeatureManifest`，被 `featureRegistry.ts` 的 `import.meta.glob` 自动收集。

---

## 3. 涉及类清单（全路径）

### 3.1 `frontend/src/features/secretary/index.tsx`
- `default export FeatureManifest`
  - `id: 'secretary'`、`name: '个人秘书'`、`icon: NotebookPen`、`group: '内容工具'`、`order: 35`
  - `routes: [{ path: '/tools/secretary', element: <SecretaryPage /> }]`

### 3.2 `frontend/src/features/secretary/types.ts`
- `type InputMethod = 'text' | 'voice' | 'file'`
- `interface EntryGeo { latitude, longitude, accuracy, capturedAt }`
- `interface BaseEntry { id, createdAt, inputMethod, geo, note? }`
- `interface TextEntry extends BaseEntry { inputMethod:'text', text }`
- `interface VoiceEntry extends BaseEntry { inputMethod:'voice', durationMs, mimeType }`
- `interface FileEntry extends BaseEntry { inputMethod:'file', fileName, fileSize, mimeType }`
- `type Entry = TextEntry | VoiceEntry | FileEntry`

### 3.3 `frontend/src/features/secretary/lib/entryRepo.ts`
- `init(): Promise<IDBDatabase>` —— 单例，懒打开；upgrade 时建 `entries`（keyPath=`id`，索引 `byCreatedAt`）和 `blobs`（keyPath=`id`）两个 store
- `list(): Promise<Entry[]>` —— 用 `byCreatedAt` 索引倒序 `openCursor(null, 'prev')` 收集
- `add(entry: Entry, blob?: Blob): Promise<void>` —— 同一事务写两个 store
- `getBlob(id: string): Promise<Blob | null>`
- `remove(id: string): Promise<void>` —— 同事务删 entries + blobs
- `clear(): Promise<void>` —— 调试用

DB 常量：`DB_NAME = 'kai-toolbox/secretary'`、`DB_VERSION = 1`。

### 3.4 `frontend/src/features/secretary/lib/createEntry.ts`
- `createEntry(input): Promise<{ entry: Entry, blob?: Blob }>`
  - input 联合类型：`{ kind:'text', text } | { kind:'voice', blob, durationMs, mimeType } | { kind:'file', file: File }`
  - 内部：`tryGetGeo()` 并行启动 → 构造 entry → 把 `blob` / `file` 转 Blob 一并返回
  - **注意**：本函数只负责构造，不调 repo；由 page 层串接，方便单测

### 3.5 `frontend/src/features/secretary/lib/geo.ts`
- `tryGetGeo(timeoutMs = 4000): Promise<EntryGeo | null>`
  - `navigator.geolocation` 不存在 → null
  - 超时 / error / 权限拒绝 → null
  - 成功 → `{ latitude, longitude, accuracy, capturedAt: Date.now() }`

### 3.6 `frontend/src/features/secretary/lib/format.ts`
- `formatDurationShort(ms: number): string` —— `mm:ss`
- `groupKeyOf(epochMs: number): string` —— `今天 / 昨天 / YYYY-MM-DD`
- `pickVoiceMime(): string` —— 协商 `audio/webm;codecs=opus` 优先，回退 `audio/webm` / `audio/mp4`

### 3.7 `frontend/src/features/secretary/components/ComposerTabs.tsx`
- Props: `value: 'text'|'voice'|'file'`、`onChange`
- 复用 `@/components/ui/segmented`（已有）

### 3.8 `frontend/src/features/secretary/components/TextComposer.tsx`
- Props: `onSubmit(text: string): Promise<void>` 、`disabled?`
- 内部维护 `text` state、回车（无 Shift）触发提交、提交后清空

### 3.9 `frontend/src/features/secretary/components/VoiceRecorder.tsx`
- Props: `onSubmit(blob, durationMs, mimeType): Promise<void>`
- 内部状态：`'idle' | 'recording' | 'submitting' | 'unsupported'`
- 用 `MediaRecorder` 收集 `dataavailable` 切片，stop 时 concat 成 Blob
- 计时用 `useEffect + setInterval(250ms)`
- 卸载时强制 `mediaRecorder?.stop()` + `stream.getTracks().forEach(t=>t.stop())`，避免麦克风占用

### 3.10 `frontend/src/features/secretary/components/AttachmentPicker.tsx`
- Props: `onSubmitFiles(files: File[]): Promise<void>`
- 隐藏 `<input ref type="file" multiple>` + 自定义按钮
- 选好后清空 `input.value`（让相同文件可重复选择）

### 3.11 `frontend/src/features/secretary/components/Timeline.tsx`
- Props: `entries: Entry[]`、`onOpen(id)`、`onRemove(id)`
- 按 `groupKeyOf(createdAt)` 分组，分组内仍按 `createdAt` 倒序
- 空状态文案：`还没有记录，随手记一条吧`

### 3.12 `frontend/src/features/secretary/components/EntryItem.tsx`
- 单条卡片；图标按 `inputMethod`（`MessageSquare` / `Mic` / `Paperclip`）
- 文字截前 200 字加省略号；语音显示时长 + 播放小按钮；附件显示 fileName + formatBytes(size)
- 地点：有 `geo` 时显示 `📍 lat, lng (±accuracy m)` 一行小字
- 删除按钮触发 `confirm-dialog`（已有 ui 组件）

### 3.13 `frontend/src/features/secretary/components/EntryDetail.tsx`
- 复用 `@/components/ui/sheet`
- 根据 inputMethod 渲染：文字全文 / `<audio controls>` / 附件下载按钮 + 图片预览（mime 以 `image/` 开头）

### 3.14 `frontend/src/features/secretary/pages/SecretaryPage.tsx`
- State：`entries: Entry[]`、`hydrated: boolean`、`composerKind: InputMethod`、`detailId: string | null`、`submitting: boolean`
- `useEffect` 首屏调 `entryRepo.list()` → setEntries
- 三个 Composer 的 `onSubmit` 统一走 `handleSubmit(kind, payload)`：
  1. `createEntry(input)` → `{ entry, blob }`
  2. `entryRepo.add(entry, blob)`
  3. `setEntries(prev => [entry, ...prev])`
  4. toast 提示成功（暂无 toast 系统，先用页内 banner 兜底）

布局：上方 `ComposerTabs` + 当前 Composer 卡片；下方 `Timeline`。移动端纵向堆叠。

---

## 4. 数据结构

### 4.1 IndexedDB

```text
DB: kai-toolbox/secretary   v1
  store entries
    keyPath = 'id'
    index byCreatedAt -> 'createdAt' (unique=false)
  store blobs
    keyPath = 'id'
```

写入 `add` 时把 entry 序列化为纯对象（不含 Blob），Blob 单独写 `blobs`，两步在同一事务里。

### 4.2 同步快照（仅在 Page 内存中）

`entries: Entry[]`，按 `createdAt desc`。新增时 `[entry, ...prev]`，删除时 `prev.filter(e => e.id !== id)`。

---

## 5. 重要约束与边界

| 约束 | 实现处 |
|------|-------|
| 时间戳 / id / inputMethod 由工厂注入 | `createEntry.ts`，禁止 Page 直接构造 entry |
| geo 永不 throw | `geo.ts` 用 `try/catch + Promise.race(timeout)` |
| Blob 永不进内存数组 | `entries` state 只存元数据，Blob 按需 `getBlob(id)` |
| 麦克风必释放 | `VoiceRecorder` 在 stop 之后 + unmount 时执行 `stream.getTracks().forEach(t=>t.stop())` |
| URL.createObjectURL 必撤销 | `EntryDetail` 在 `useEffect cleanup` 调 `URL.revokeObjectURL` |
| 删除原子化 | `entryRepo.remove` 同事务删两个 store；任一失败回滚 |
| 注释语言 | kai-toolbox 注释一律中文（按用户偏好） |
| 触屏 Enter 不提交 | `TextComposer` 用 `matchMedia('(pointer: coarse)')` 判定，触屏设备 Enter 仅换行 |
| Composer 可折叠 | `SecretaryPage` 顶部 sticky，`composerOpen` 控制 max-height 过渡 |
| 详情抽屉小屏全宽 | `EntryDetail` 给 `SheetContent` 加 `!max-w-full sm:!max-w-md` 覆盖 |
| 移动端字号 ≥16px | textarea `text-base sm:text-sm`，避免 iOS Safari 聚焦自动缩放 |

---

## 6. 测试 / 验证要点

代码不含自动化测试（与现有 feature 一致）；通过手工验证：

1. `npm run typecheck` 通过
2. `npm run dev` 启动后访问 `/tools/secretary`
3. 三种录入分别走一遍，刷新后数据仍在
4. 拒绝地理位置权限后再次录入，主流程不阻塞
5. 删除一条带语音的 entry，IDB devtools 中 `blobs` 也消失
