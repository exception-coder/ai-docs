# 局域网文件传输 - 空间化交互改版（编码摘要）

> 本文档对应：`局域网文件传输-空间化交互改版-current.md`
>
> **职责边界**：设计文档回答"为什么这样做、核心架构、风险"；本文档回答"每个组件 / hook / service 的方法怎么写"。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| current | 2026-05-05 | 初版：与设计文档同步 |

---

## 1. 核心业务规则

- 点击自身 DeviceAvatar 不触发 ActionSheet；只允许长按本机弹出"修改设备类型"面板。
- 用户手动覆盖的 kind 持久化到 `localStorage['lan-share.deviceKind']`；自动识别结果不持久化。
- 未收到对端 `device-profile` 控制消息时，DeviceAvatar 渲染 `kind = 'unknown'` 通用图标，**不阻塞**文件传输。
- ActionSheet 打开期间不允许重复点选其他设备（单一焦点）。
- 群发包含离线/未握手 peer 时，PCM 内部按 peer 维度分别处理失败，UI 不阻塞其他 peer。
- 设备数 = 0（仅自己）时隐藏 BroadcastButton；本机 DeviceAvatar 居中显示"等待加入"文字提示。
- 屏宽 < 360px 自动降级为竖向列表卡，放弃等距投影，但保留拟物图标。
- 设备数 ≤ 6 用环形布局；≥ 7 切换为 3 列网格。
- `prefers-reduced-motion: reduce` 时关闭 SVG 粒子动效，仅保留静态连线。
- Mock 模式下 useDeviceProfileExchange 走 mockOrchestrator 路径，不发真实 control 消息；为虚拟 peer 各分配一个固定 DeviceKind。
- 用户长按本机改 kind 后必须立即调用 `useDeviceProfileExchange.broadcast()`，对端在 < 1s 内看到图标更新。
- DeviceAvatar 触摸点击区域 ≥ 56x56 px。

---

## 2. 接口入口指针

> 本次纯前端，无 HTTP 接口新增。已有信令 / ICE 接口不变。

| 入口 | 类型 | 实现位置 |
|------|------|---------|
| 页面挂载 | React 组件 | `frontend/src/features/lan-share/pages/LanSharePage.tsx`（不变） |
| 房间视图 | React 组件 | `frontend/src/features/lan-share/components/RoomView.tsx`（修改：壳层精简） |
| 房间场景 | React 组件 | `frontend/src/features/lan-share/components/scene/RoomScene.tsx`（新增） |
| WebSocket 信令 | 既有 | `wss://host/api/lan-share/signaling` |
| ICE 配置 | 既有 | `GET /api/lan-share/ice-config` |
| 设备画像广播 | DataChannel control 消息 | `ControlMessage` 联合追加 `{ type: 'device-profile'; profile }` |

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `frontend/src/features/lan-share/types.ts` | 修改 | 新增 `DeviceKind` 联合类型；新增 `DeviceProfile` 接口；`ControlMessage` 联合追加 `device-profile` |
| `frontend/src/features/lan-share/services/deviceProfile.ts` | 新建 | UA 解析 + localStorage 覆盖 |
| `frontend/src/features/lan-share/services/peerConnectionManager.ts` | 修改 | 在 control 消息分发处加 `case 'device-profile'`；暴露 `broadcastControl` 公共方法 |
| `frontend/src/features/lan-share/services/mockOrchestrator.ts` | 修改 | mock peers 分配 DeviceKind；模拟 device-profile 同步 |
| `frontend/src/features/lan-share/hooks/useRoom.ts` | 修改 | 增加 control 消息回调路由；暴露 `onDeviceProfile` 订阅入口 |
| `frontend/src/features/lan-share/hooks/useDeviceProfileExchange.ts` | 新建 | 本机 profile 广播 + 远端 profile 缓存 |
| `frontend/src/features/lan-share/hooks/useDeviceLayout.ts` | 新建 | 屏幕自适应坐标计算 |
| `frontend/src/features/lan-share/hooks/useSceneInteractions.ts` | 新建 | 点击 / 长按 / 选文件编排 |
| `frontend/src/features/lan-share/components/RoomView.tsx` | 修改 | 移除 PeerList + FileSender，挂载 RoomScene |
| `frontend/src/features/lan-share/components/scene/RoomScene.tsx` | 新建 | 等距 2.5D 场景容器 |
| `frontend/src/features/lan-share/components/scene/DeviceAvatar.tsx` | 新建 | 单设备拟物 SVG + 状态光晕 + 进度环 |
| `frontend/src/features/lan-share/components/scene/DeviceKindIcon.tsx` | 新建 | DeviceKind 到 SVG 内联组件的映射 |
| `frontend/src/features/lan-share/components/scene/ConnectionLayer.tsx` | 新建 | SVG 连线 + 粒子动效层 |
| `frontend/src/features/lan-share/components/scene/BroadcastButton.tsx` | 新建 | 中央广播按钮 |
| `frontend/src/features/lan-share/components/scene/TargetActionPanel.tsx` | 新建 | ActionSheet/Drawer 内容组件 |
| `frontend/src/features/lan-share/components/scene/styles.module.css` | 新建 | 等距视角 transform + 呼吸光晕 keyframes |
| `frontend/src/features/lan-share/components/IncomingDialog.tsx` | 修改 | 顶部加发送方 kind 图标（弱视觉调整） |
| `frontend/src/features/lan-share/components/TransferList.tsx` | 修改 | 列表项左侧加 peer kind 小图标 |
| `frontend/src/features/lan-share/components/PeerList.tsx` | 删除 | 由 RoomScene + DeviceAvatar 取代 |
| `frontend/src/features/lan-share/components/FileSender.tsx` | 删除 | 由 ActionSheet + BroadcastButton 取代 |
| `frontend/src/features/lan-share/mock.ts` | 修改 | mock peers 标注 deviceKind |

### 关键方法签名与职责

#### `services/deviceProfile.ts`（新建）

```ts
// 模块级常量
const STORAGE_KEY = 'lan-share.deviceKind'

// 类型推断（暴露）
detectDeviceKind(): DeviceKind
  — 优先读 navigator.userAgentData.platform + mobile，兜底 navigator.userAgent + navigator.platform。
  — 顺序：iOS Phone → iPad → Android Phone → Android Tablet → Windows → Mac → Linux → unknown。

getOverride(): DeviceKind | null
  — 从 localStorage[STORAGE_KEY] 读取；非合法 DeviceKind 时返回 null。

setOverride(kind: DeviceKind): void
  — 写入 localStorage；触发同窗口内的回调（用 BroadcastChannel 或 storage 事件，按需，最简：直接由调用方在调用后通知）。

clearOverride(): void
  — localStorage.removeItem。

getDeviceProfile(): DeviceProfile
  — 合并：override > detect。返回 { kind, modelHint, colorHint }，modelHint/colorHint 当前版本留空。
```

#### `services/peerConnectionManager.ts`（修改）

```ts
// 已有 sendControl(peerId, msg)；新增对外的批量广播方法
broadcastControl(peers: Peer[], msg: ControlMessage): void
  — 对每个已建立 DataChannel 的 peer 调 sendControl，未建立的跳过；不抛出。

// 内部 onMessage 分支已有 offer/accept/reject/progress/complete/cancel；追加：
case 'device-profile':
  callbacks.onDeviceProfile?.(peer, msg.profile)
  break

// PeerConnectionCallbacks 接口追加可选项：
onDeviceProfile?: (peer: Peer, profile: DeviceProfile) => void
```

#### `hooks/useRoom.ts`（修改）

```ts
// 已有 UseRoomResult 接口追加：
deviceProfiles: Map<string, DeviceProfile>      // deviceId → profile
broadcastSelfProfile: () => void                 // 触发主动重广播
setLocalProfileOverride: (kind: DeviceKind | null) => void

// 内部状态追加：
const [deviceProfiles, setDeviceProfiles] = useState<Map<string, DeviceProfile>>(new Map())

// PeerConnectionManager 创建时注入 onDeviceProfile：
onDeviceProfile: (peer, profile) => {
  setDeviceProfiles(prev => new Map(prev).set(peer.deviceId, profile))
}

// peer-joined 信令到达后：补发本机 profile 给新人
signaling.on('peer-joined', msg => {
  setPeers(prev => ...)
  // 等 PCM 与新人握手成功后由 useDeviceProfileExchange 主动广播
})
```

#### `hooks/useDeviceProfileExchange.ts`（新建）

```ts
type Args = {
  selfDeviceId: string
  peers: Peer[]
  pcm: PeerConnectionManager | null
  enabled: boolean         // mock 模式时为 false，由外部直接灌注
}

useDeviceProfileExchange(args: Args): {
  selfProfile: DeviceProfile
  broadcastSelf: () => void
  setSelfKindOverride: (kind: DeviceKind | null) => void
}

// 内部行为：
// 1. mount 时读取本机 profile（detectDeviceKind + getOverride）
// 2. 监听 peers 列表变化：对每个新 peer，等其 DataChannel open 后调用 pcm.sendControl(peerId, { type: 'device-profile', profile: selfProfile })
// 3. 暴露 broadcastSelf：对所有当前 peer 重发；setSelfKindOverride 触发持久化 + broadcastSelf
// 4. enabled = false 时所有方法变 no-op
```

#### `hooks/useDeviceLayout.ts`(新建)

```ts
type Args = {
  peerCount: number      // 含自己
  viewport: { w: number; h: number }
}

type Position = { x: number; y: number; scale: number; isSelf: boolean }

useDeviceLayout(args: Args): {
  layout: 'isometric-circle' | 'isometric-grid' | 'vertical-list'
  positions: Position[]   // 索引 0 = 本机，其余按 peers 顺序
}

// 算法：
// - viewport.w < 360: layout = vertical-list；纯纵向堆叠
// - peerCount <= 6: isometric-circle；本机居中或置底，其他等角度环绕
// - peerCount >= 7: isometric-grid；3 列 N 行
```

#### `hooks/useSceneInteractions.ts`（新建）

```ts
type Args = {
  selfDeviceId: string
  sendFileTo: (peerDeviceId: string, file: File) => void
  broadcastFile: (file: File) => void
}

useSceneInteractions(args: Args): {
  selectedTarget: Peer | null
  isPanelOpen: boolean
  selectTarget: (peer: Peer) => void
  closePanel: () => void
  pickFileAndSend: () => Promise<void>     // 触发隐藏 file input，选定后调 sendFileTo
  pickFileAndBroadcast: () => Promise<void>
}

// 行为：
// - selectTarget(peer): if peer.deviceId === selfDeviceId, no-op；否则 setSelectedTarget + isPanelOpen=true
// - closePanel: isPanelOpen=false，保留 selectedTarget 直到下次选中
// - pickFileAndSend: 内部维护一个 hidden <input>，change 时调 sendFileTo(selectedTarget.deviceId, file)，并 closePanel
```

#### `components/scene/RoomScene.tsx`（新建）

```ts
type Props = {
  selfDeviceId: string
  selfNickname: string
  selfProfile: DeviceProfile
  peers: Peer[]
  deviceProfiles: Map<string, DeviceProfile>
  transfers: Transfer[]
  onSendFileTo: (peerDeviceId: string, file: File) => void
  onBroadcastFile: (file: File) => void
  onChangeSelfKind: (kind: DeviceKind) => void
}

// 行为：
// 1. useDeviceLayout 计算坐标
// 2. useSceneInteractions 维护选中 / 打开面板
// 3. 渲染：
//    - 容器 div 应用 isometric transform（layout != vertical-list 时）
//    - peers + 本机 各渲染一个 DeviceAvatar，绑定 onClick / onLongPress
//    - ConnectionLayer 渲染所有 transferring transfer 的连线
//    - BroadcastButton 居中浮动
//    - TargetActionPanel 由 selectedTarget 驱动
//    - 长按本机弹出 DeviceKindPicker（简易选择列表）
```

#### `components/scene/DeviceAvatar.tsx`(新建)

```ts
type Props = {
  kind: DeviceKind
  nickname: string
  position: Position
  state: 'idle' | 'connecting' | 'connected' | 'transferring' | 'failed'
  progress?: number      // 0..1，仅 transferring 时有效
  isSelf: boolean
  onClick?: () => void
  onLongPress?: () => void
}

// 行为：
// 1. 用 absolute 定位 + transform: translate(x, y) scale(s)
// 2. 内部包含：
//    - <DeviceKindIcon kind={kind} />（拟物 SVG）
//    - 状态光晕：CSS class 切换（.state-connecting/.state-failed/...）
//    - 进度环：state === 'transferring' 时圆形 SVG stroke-dasharray 动画
//    - 昵称文字标签下方
// 3. 触摸事件：
//    - onClick：单击触发
//    - onLongPress：onPointerDown + setTimeout(500ms)；onPointerUp/Move 取消
// 4. 点击区域 ≥ 56x56 px（含透明 padding）
```

#### `components/scene/DeviceKindIcon.tsx`（新建）

```ts
type Props = { kind: DeviceKind; size?: number }

// 内部 switch(kind) 渲染对应内联 SVG：
// - iphone: 圆角矩形 + 顶部刘海条 + 底部 Home Indicator
// - ipad: 横版圆角矩形 + 摄像头点
// - android-phone: 圆角矩形 + 顶部水滴 + 底部三键
// - android-tablet: 横版圆角矩形 + 横向摄像头
// - windows: 显示器 + 底座 + 任务栏方块
// - mac: 笔记本翻盖（盖 + 底座）+ 苹果光晕
// - linux: 圆形头像 + 企鹅剪影
// - unknown: 通用方框 + 问号
```

#### `components/scene/ConnectionLayer.tsx`(新建)

```ts
type Props = {
  selfPosition: Position
  peerPositions: Map<string, Position>
  transfers: Transfer[]      // 来自 useRoom
}

// 行为：
// 1. 顶层 SVG，width/height = 100%，pointer-events: none
// 2. 对每个 state in (pending|transferring|failed) 的 transfer：
//    - 计算 self ↔ peer 的两点
//    - 渲染 <path d="M x1 y1 L x2 y2"> ；class 按 state 切换
//    - state === 'transferring': 套 3 个 <circle><animateMotion> 沿 path 移动
//    - state === 'failed': stroke=red，dasharray，1 次抖动 keyframes
//    - state === 'completed': 1.2s 内淡出后 unmount（由父组件控制 transfer 列表）
// 3. prefers-reduced-motion: reduce 时不挂 animateMotion
```

#### `components/scene/BroadcastButton.tsx`（新建）

```ts
type Props = {
  peerCount: number
  onPickFile: () => void
}

// 行为：
// - peerCount === 0：disabled，文字"等待其他设备加入"
// - 否则：圆形大按钮 + 中心 Send 图标，点击调用 onPickFile
// - 内部不持有 file input；由 useSceneInteractions 提供
```

#### `components/scene/TargetActionPanel.tsx`（新建）

```ts
type Props = {
  open: boolean
  target: Peer | null
  targetProfile: DeviceProfile | undefined
  onClose: () => void
  onPickFile: () => void
}

// 行为：
// - 通过 CSS media query：viewport < 768 渲染底部 ActionSheet（fixed bottom，slide-up）；否则右侧 Drawer
// - 内容：DeviceKindIcon + 昵称 + "选择文件并发送"按钮 + "关闭"按钮
// - open=false 时返回 null
```

---

## 4. 数据结构

### `types.ts` 新增/修改

```ts
// 新增
export type DeviceKind =
  | 'iphone'
  | 'ipad'
  | 'android-phone'
  | 'android-tablet'
  | 'windows'
  | 'mac'
  | 'linux'
  | 'unknown'

export interface DeviceProfile {
  kind: DeviceKind
  modelHint?: string     // 当前版本留空，未来扩展
  colorHint?: string     // 当前版本留空，未来扩展
}

// 修改：ControlMessage 联合追加一员
export type ControlMessage =
  | { type: 'offer'; fileId: string; name: string; size: number; mime?: string; totalChunks: number }
  | { type: 'accept'; fileId: string }
  | { type: 'reject'; fileId: string; reason?: string }
  | { type: 'progress'; fileId: string; received: number }
  | { type: 'complete'; fileId: string }
  | { type: 'cancel'; fileId: string }
  | { type: 'device-profile'; profile: DeviceProfile }      // 新增
```

### localStorage

```
key: lan-share.deviceKind
value: DeviceKind 字符串（如 'iphone'）
不合法值或不存在：detectDeviceKind() 自动识别
```

---

## 5. 重要约束与边界

- **协议向后兼容**：旧版客户端收到 `device-profile` control 消息时落入 `default` 分支（既有代码已有对未知 type 的兜底，本次需确认 PCM 内 default 分支是 `console.warn` 不是 throw）。
- **DataChannel 时机**：新 peer 加入时，必须等 DataChannel `open` 才发送 device-profile，否则丢失。useDeviceProfileExchange 监听 `peer-connection-ready` 类事件（PCM 已暴露则用，否则需新增 onChannelOpen 回调）。
- **Mock 模式互不干扰**：`isMockEnabled()` 时 useDeviceProfileExchange 不挂 PCM 监听；mockOrchestrator 内部直接构造 profileMap 并调用 setState 接口（暴露注入点）。
- **性能**：DeviceAvatar 重渲染避免昂贵计算；状态切换走 className，不重新生成 SVG。
- **不处理**：peer 短时间内反复进出房间的 profile 抖动（直接以最后一次为准，不去重）。

---

## 6. 下游依赖调用

```
浏览器 API：
- navigator.userAgentData (Chromium only)
- navigator.userAgent (兜底)
- navigator.platform
- window.matchMedia('(prefers-reduced-motion: reduce)')
- window.matchMedia('(max-width: 767px)')

既有内部依赖：
- services/peerConnectionManager.ts # sendControl / broadcastControl
- services/signalingClient.ts (不直接调，沿用 useRoom)
- services/identity.ts # getOrCreateDeviceId（不变）
```

---

## 7. 异常处理要点

- `detectDeviceKind`：UA 解析失败 → 返回 `unknown`，不抛错。
- `getOverride`：JSON 损坏或非法 kind → 返回 null，并 `localStorage.removeItem`。
- `useDeviceProfileExchange.broadcastSelf`：单个 peer 发送失败 → `console.warn` 不阻塞其他 peer。
- `ConnectionLayer`：position 缺失（peer 未在 layout 中）→ 跳过该连线。
- `RoomScene` 长按本机切换 kind：选错也只是视觉变化，不影响传输；用户可再切。
- `prefers-reduced-motion: reduce`：强制走静态线，不需要降级提示。
- 屏宽变化（旋转）：useDeviceLayout 在 useEffect 内监听 resize，节流 200ms 重新计算。
