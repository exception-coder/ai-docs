# Markdown 文本转卡片 · 编码摘要

> 对应设计文档：`Markdown文本转卡片-current.md`
> **职责边界**：本文档只回答"每个文件 / 方法怎么写"，业务规则与架构图由 current.md 承载。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-05-06 | 初版：实现三模式 + 5 主题 + 跨端导出 + localStorage 持久化 |
| v2 | 2026-05-06 | 幻灯模式新增 SplitMode（manual / paragraph） |

---

## 1. 核心业务规则

- 模式与主题正交：切换任意一维不重置另一维与文本
- 三种模式：`xiaohongshu`（750×自适应）/ `slide`（16:9 或 9:16，默认 16:9）/ `preview`（响应式）
- 5 套主题：`minimal / dark / xiaohongshu / zhihu / terminal`，CSS 用 `[data-md-theme="xxx"]` 选择器实现
- markdown 渲染必须经 DOMPurify 清洗，禁止 raw script
- 多页切片 `manual`：行级 `^---\s*$`，空白页跳过，至少 1 页
- 多页切片 `h1`（默认）：遇 `^#\s` 行开新页，适合一级章节内容多的文章
- 多页切片 `h1h2`：遇 `^#{1,2}\s` 行开新页，适合二级章节粒度
- 状态持久化 `splitMode` 字段默认 `'h1'`
- 导出前 `await document.fonts.ready`，避免中文回退
- 导出文件名：`md-card-{mode}-{yyyymmddHHmmss}.png`，多页加 `-{页码}`
- 导出降级链：`navigator.share` (with files) → `<a download>` → 新窗口长按
- 状态持久化：sourceText / mode / theme / slideRatio / watermark 写入 localStorage `kai-toolbox:markdown-card:state`
- 大文本保护：sourceText > 50000 时停用实时渲染，改为按钮触发

---

## 2. 接口入口指针

> 纯前端模块，无对外 HTTP 接口。

| 入口 | 实现 |
|------|------|
| 路由 `/tools/markdown-card` | `frontend/src/features/markdown-card/pages/MarkdownCardPage.tsx` |
| 注册 manifest | `frontend/src/features/markdown-card/index.tsx` |

---

## 3. 涉及文件清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `frontend/package.json` | 修改 | 新增 `marked`, `dompurify`, `@types/dompurify`, `html-to-image` |
| `frontend/src/features/markdown-card/index.tsx` | 新建 | feature manifest，注册到 featureRegistry |
| `frontend/src/features/markdown-card/types.ts` | 新建 | Mode / Theme / SlideRatio / Watermark / PersistedState 类型 |
| `frontend/src/features/markdown-card/pages/MarkdownCardPage.tsx` | 新建 | 顶层容器，唯一状态源 |
| `frontend/src/features/markdown-card/components/MarkdownEditor.tsx` | 新建 | 受控 textarea + 字数统计 |
| `frontend/src/features/markdown-card/components/ModeSwitcher.tsx` | 新建 | 三模式分段控件 |
| `frontend/src/features/markdown-card/components/ThemeSelector.tsx` | 新建 | 主题分段控件（5 选项） |
| `frontend/src/features/markdown-card/components/SlideRatioSwitcher.tsx` | 新建 | 16:9 / 9:16 切换（仅 slide 模式可见） |
| `frontend/src/features/markdown-card/components/WatermarkForm.tsx` | 新建 | 水印三段配置（仅 xiaohongshu 模式可见） |
| `frontend/src/features/markdown-card/components/ExportButton.tsx` | 新建 | 触发导出，支持单图 / 多页批量 |
| `frontend/src/features/markdown-card/components/CardRenderer.tsx` | 新建 | 按 mode 分发到子渲染器 |
| `frontend/src/features/markdown-card/components/XiaohongshuCard.tsx` | 新建 | 750×N 竖版长图 + 水印 |
| `frontend/src/features/markdown-card/components/SlideCards.tsx` | 新建 | 多页幻灯 + 翻页器 + 暴露所有页 ref |
| `frontend/src/features/markdown-card/components/PreviewCard.tsx` | 新建 | 响应式纯预览 |
| `frontend/src/features/markdown-card/lib/markdownPipeline.ts` | 新建 | parse + sanitize + splitSlides |
| `frontend/src/features/markdown-card/lib/themes.ts` | 新建 | 主题枚举 + 元数据 + getThemeAttr |
| `frontend/src/features/markdown-card/lib/exporter.ts` | 新建 | captureNode + saveImage + exportSlides |
| `frontend/src/features/markdown-card/lib/persistence.ts` | 新建 | localStorage 读写 + 容错 |
| `frontend/src/features/markdown-card/styles/card-themes.css` | 新建 | 5 套主题 CSS 变量 |

### 关键方法签名与职责

```ts
// types.ts
type Mode = 'xiaohongshu' | 'slide' | 'preview'
type Theme = 'minimal' | 'dark' | 'xiaohongshu' | 'zhihu' | 'terminal'
type SlideRatio = '16:9' | '9:16'
interface Watermark { signature: string; subSignature: string; qrcodeUrl: string }
interface PersistedState { sourceText: string; mode: Mode; theme: Theme; slideRatio: SlideRatio; watermark: Watermark }

// markdownPipeline.ts
parseMarkdown(text: string): string                    // marked + DOMPurify, 返回 sanitized HTML
splitSlides(text: string): string[]                    // 行级 --- 切片, 至少返回长度 1 的数组（空白页过滤）

// themes.ts
THEMES: ReadonlyArray<{ id: Theme; label: string }>    // UI 渲染用
getThemeAttr(theme: Theme): { 'data-md-theme': Theme }  // 用于挂在卡片根 DOM

// persistence.ts
loadState(): PersistedState                             // 失败/空时返回 DEFAULT_STATE
saveState(state: PersistedState): void                  // try/catch 包裹，失败仅 console.warn
DEFAULT_STATE: PersistedState                           // 导出常量

// exporter.ts
captureNode(node: HTMLElement, scale?: number): Promise<string>      // toPng + await fonts.ready, 返回 dataURL
saveImage(dataUrl: string, filename: string): Promise<'shared'|'downloaded'|'fallback'>  // 三级降级
exportSlides(nodes: HTMLElement[], baseName: string, onProgress?: (i:number,total:number)=>void): Promise<void>  // 串行导出多页
buildFilename(mode: Mode, page?: number): string        // md-card-{mode}-{ts}[-{page}].png

// MarkdownCardPage.tsx
useMarkdownCardState()                                  // 自定义 hook：装载 PersistedState + 自动 saveState
handleExport()                                          // 单图导出（xhs / preview）
handleExportSlides()                                    // 多页批量导出（slide）
```

---

## 4. 数据结构

### localStorage 键

```
kai-toolbox:markdown-card:state
{
  "sourceText": "# Hello\n",
  "mode": "preview",
  "theme": "minimal",
  "slideRatio": "16:9",
  "watermark": { "signature": "", "subSignature": "", "qrcodeUrl": "" }
}
```

### 主题 CSS 变量（每个 `[data-md-theme="xxx"]` 块）

```
--md-card-bg
--md-card-fg
--md-card-heading
--md-card-subheading
--md-card-link
--md-card-quote-bg
--md-card-quote-border
--md-card-code-bg
--md-card-code-fg
--md-card-divider
--md-card-watermark
--md-card-font-family
--md-card-font-mono
```

### 卡片尺寸（逻辑像素，导出 2x）

| 模式 | 宽 | 高 |
|------|---|---|
| xiaohongshu | 750 | 自适应（最少 1000） |
| slide 16:9 | 1280 | 720 |
| slide 9:16 | 720 | 1280 |
| preview | 100% | auto |

---

## 5. 重要约束与边界

- **状态唯一来源**：`MarkdownCardPage` 持有所有状态，子组件全部受控；禁止子组件 `useState` 自管业务状态
- **DOM ref 暴露**：`XiaohongshuCard` / `PreviewCard` forwardRef 暴露根节点；`SlideCards` 通过 imperative handle 暴露 `getSlideNodes(): HTMLElement[]`
- **导出 DOM 不变形**：导出前不要给节点加 `transform: scale`，由 `html-to-image` 的 `pixelRatio` 处理高清
- **DOMPurify 配置**：`{ USE_PROFILES: { html: true } }`，默认禁脚本；如需图片白名单后续再开
- **localStorage 容错**：`JSON.parse` 抛错时回退默认；写入抛错（隐私模式）时仅 `console.warn`
- **大文本阈值**：`sourceText.length > 50000` 时 `MarkdownEditor` 不再实时 onChange 触发渲染，改为"刷新预览"按钮 commit；`SourceText` 仍正常受控
- **首页排序**：manifest `order: 30`（位于现有 20-tier 之后），`group: '内容工具'`
- **不引入 file-saver**：直接 `URL.createObjectURL + a.click + URL.revokeObjectURL`，少一个依赖

---

## 6. 下游依赖调用

```
// 第三方库
import { marked } from 'marked'
import DOMPurify from 'dompurify'
import { toPng } from 'html-to-image'

// 浏览器 API
navigator.canShare?.({ files: [file] })
navigator.share?.({ files: [file] })
URL.createObjectURL(blob)
document.fonts.ready
```

---

## 7. 异常处理要点

- `marked.parse` 抛错（极少）→ 渲染区显示 "Markdown 解析失败：{message}"，不阻塞编辑
- `DOMPurify` 缺失 → 编译期 fail-fast，运行期不发生
- `toPng` 抛错 → ExportButton 切回常规态，弹 toast/inline 错误："导出失败，请重试"
- `navigator.share` rejected（用户取消）→ 静默吞掉，不报错；rejected 且非 `AbortError` → 走 `<a download>` 兜底
- `<a download>` 不可用（极旧浏览器）→ `window.open(dataUrl)` 让用户长按保存
- `localStorage.setItem` 抛 QuotaExceededError → `console.warn`，仍维持内存中状态
- 空文本导出 → ExportButton 禁用，不允许触发
- `sourceText` 仅含空白：渲染区显示空状态，导出按钮禁用
