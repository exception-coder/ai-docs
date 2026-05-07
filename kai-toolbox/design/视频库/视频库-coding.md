# 视频库 — 编码摘要

> 配套：`视频库-current.md`

## 1. 核心规则速查

- 视频清单：`treesize_node` `is_dir=0` + 扩展名命中后端白名单 + 所属 scan `status='COMPLETED'`
- 排序：仅 `name` / `size` × `asc` / `desc`，非法值默认 `name asc`
- LIMIT 5000；超限 UI 提示
- 切换视频靠 React `key` 重渲染 `<VideoPlayer>`，旧实例 cleanup 触发后端 ffmpeg 清理
- 移动 < md 单列 + 底部 Sheet；桌面 ≥ md 双列 sidebar；按钮最小 48×48

## 2. 接口入口

| Method | Path | 实现类#方法 |
|--------|------|-------------|
| GET | `/api/treesize/videos?sortBy=&order=` | `TreeSizeController#libraryVideos` |

入参：`sortBy ∈ {name,size}` 默认 name；`order ∈ {asc,desc}` 默认 asc。

返回：`List<VideoLibraryItemView>`，含 `scanId / rootPath / path / name / size`。

## 3. 涉及类

### 后端

```java
// 包：com.exceptioncoder.toolbox.treesize.api.dto
public record VideoLibraryItemView(String scanId, String rootPath, String path, String name, long size) {}

// 包：com.exceptioncoder.toolbox.treesize.repository
public class NodeRepository {
    // 已有方法 + 新增：
    public List<NodeRow> findVideos(List<String> extensions, String sortBy, String order, int limit);
    // 用 SQL JOIN treesize_scan 拿 root_path 一并返回。
}

// 包：com.exceptioncoder.toolbox.treesize.api
public class TreeSizeController {
    // 新增：
    @GetMapping("/videos")
    public List<VideoLibraryItemView> libraryVideos(
        @RequestParam(defaultValue = "name") String sortBy,
        @RequestParam(defaultValue = "asc") String order);
}
```

### 前端

```ts
// features/video-playback/VideoPlayer.tsx  [新建，抽自 VideoPlayerModal 内核]
interface Props {
  scanId: string
  path: string
  className?: string
}
// 内部：probe → native/hls/unsupported/error → setup video + hls.js → cleanup on unmount

// features/treesize/components/VideoPlayerModal.tsx  [改造]
// 改成：DialogPrimitive 容器 + 标题 + 关闭按钮 + <VideoPlayer/>

// features/video-library/index.tsx
// FeatureManifest: id='video-library', name='视频库', icon=Film, group='媒体', order=20

// features/video-library/types.ts
export interface VideoLibraryItem { scanId, rootPath, path, name, size }
export type VideoSortBy = 'name' | 'size'
export type VideoSortOrder = 'asc' | 'desc'

// features/video-library/api.ts
export function getVideoLibrary(sortBy: VideoSortBy, order: VideoSortOrder): Promise<VideoLibraryItem[]>

// features/video-library/pages/VideoLibraryPage.tsx
// state: { sortBy, order, selectedIndex, listOpen }
// useQuery(['video-library', sortBy, order])
// 切换排序：保留当前播放路径，重新计算索引
// < md：渲染 player 全宽 + Sheet 抽屉
// ≥ md：grid-cols-[320px_1fr]

// features/video-library/components/VideoListPanel.tsx
// props: items, selectedPath, sortBy, order, onSelect, onSortChange
// 排序选择器 + scrollable list

// features/video-library/components/VideoPlayerPanel.tsx
// props: item, onPrev, onNext, onOpenList, hasPrev, hasNext
// 渲染：<VideoPlayer key={item.path} /> + 大按钮一排（上一首 / 列表 / 下一首）+ 当前文件名
// 桌面挂 keydown ← / →
```

## 4. 关键约束

- `findVideos` 用 SQL `LIKE` 多 OR 拼接（per-extension），全部 `lower(name) LIKE lower('%.<ext>')`；`extensions` 列表来自 `VideoExtensionsProperties`，**不要硬编码**。
- 不返回扫描状态非 COMPLETED 的 scan 的节点。
- 切换视频时 React key = `item.path`，确保旧实例彻底卸载（cleanup → 后端 ffmpeg destroy）。
- VideoPlayer 不持任何业务态：纯播放器，不知道"prev/next"是什么。
- iOS Safari 走 hls.js 内置 fallback：`!Hls.isSupported() && video.canPlayType('application/vnd.apple.mpegurl')` → `video.src = playlistUrl`，已在现有 modal 代码里写过，抽出去时保留。
- 文件已被外部删除：`<VideoPlayer>` 内部 probe 收到 404 → mode='error'，UI 显示文件不存在。库页面不主动跳下一首（避免静默乱跳），按钮还是用户自己点。
- bugfix-coding-style：所有新代码不留变更日志注释。
