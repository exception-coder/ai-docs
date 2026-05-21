# JSON 格式化大文件优化 · 编码摘要

> 配套 [JSON格式化大文件优化-current.md](./JSON格式化大文件优化-current.md)
> 最后更新：2026-05-17

## 1. 核心业务规则

- 4 种操作：format / minify / escape / unescape，规则与 `lib/json.ts` 当前实现等价
- 空输入返回空字符串
- format 走所选 indent（`2` / `4` / `'\t'`）；其它 op 忽略 indent
- escape = `JSON.stringify(input)`（输入按字符串值处理）
- unescape：trim 后若不带 `"` 自动补一对；结果非字符串报「反转义结果不是字符串」
- 大输出（> 8 MB）禁用复制，仅留下载
- busy 期间按钮 disabled，避免重入

## 2. 接口入口指针

UI 入口 → Worker 端实现：

| UI 触发 | 触发函数 | Worker op |
|--------|---------|-----------|
| 「格式化」 | `frontend/src/features/formatter/components/JsonPanel.tsx#handleFormat` | `format` |
| 「压缩」 | 同上 `#handleMinify` | `minify` |
| 「转义」 | 同上 `#handleEscape` | `escape` |
| 「反转义」 | 同上 `#handleUnescape` | `unescape` |

## 3. 涉及类清单（全路径）

### 新增

| 文件 | 主要符号 | 职责 |
|------|---------|------|
| `frontend/src/features/formatter/lib/json-worker.ts` | `self.onmessage = (e) => {...}`<br/>内部 `handle(req): Res` | Worker 实体；按 `op` 分发到 `JSON.parse/stringify` 计算 |
| `frontend/src/features/formatter/lib/useJsonWorker.ts` | `useJsonWorker(): { run, busy }`<br/>`run(req: WorkerReq): Promise<WorkerRes>` | Hook：懒创建单例 Worker、id 关联回调、unmount terminate；fallback 到 `lib/json.ts` 同步实现 |
| `frontend/src/features/formatter/components/JsonEditor.tsx` | `forwardRef<JsonEditorRef, Props>`<br/>`JsonEditorRef = { getValue, setValue, focusError }` | CodeMirror 包装；uncontrolled 大文本；亮暗主题；onUpdate debounce 上报字节数 |
| `frontend/src/lib/useIsDarkTheme.ts` | `useIsDarkTheme(): boolean` | 抽自 `browser-request/pages/BrowserRequestPage.tsx`；后续两处共用 |

### 修改

| 文件 | 改动 |
|------|------|
| `frontend/src/features/formatter/components/JsonPanel.tsx` | 去除 `input` state，改成 `useRef<JsonEditorRef>`；按钮 handler 走 `useJsonWorker`；输出区改为只读 `<JsonEditor readOnly />` + 复制 / 下载按钮 |
| `frontend/src/features/browser-request/pages/BrowserRequestPage.tsx` | `useIsDarkTheme` 改 import 路径到共享 `@/lib/useIsDarkTheme`；移除本文件内的本地定义 |

### 保留（兼容 fallback）

| 文件 | 用途 |
|------|------|
| `frontend/src/features/formatter/lib/json.ts` | 纯同步实现，被 `useJsonWorker` 在 Worker 失败时调用 |

## 4. 数据结构

```ts
// frontend/src/features/formatter/lib/json-worker.ts

export type WorkerReq =
  | { id: number; op: 'format'; input: string; indent: number | '\t' }
  | { id: number; op: 'minify'; input: string }
  | { id: number; op: 'escape'; input: string }
  | { id: number; op: 'unescape'; input: string }

export type WorkerRes =
  | { id: number; ok: true;  output: string }
  | { id: number; ok: false; error: string; errorPos?: number }
```

```ts
// frontend/src/features/formatter/components/JsonEditor.tsx

export interface JsonEditorRef {
  getValue: () => string
  setValue: (v: string) => void
  /** 把光标 / 选区跳到错误位置 */
  focusError: (pos: number) => void
}

interface JsonEditorProps {
  defaultValue?: string
  readOnly?: boolean
  placeholder?: string
  minHeight?: string
  /** debounce 200ms 上报当前文本字节长度 */
  onBytesChange?: (bytes: number) => void
}
```

## 5. 重要约束与边界

- **uncontrolled 编辑器**：不要把编辑器内容当受控 prop 传入。父组件读值靠 `ref.getValue()`，写值靠 `ref.setValue(v)`。这一点是大文本性能的关键，破坏掉就回到老 textarea 卡顿状态。
- **Worker module type**：`new Worker(new URL('./json-worker.ts', import.meta.url), { type: 'module' })` 必须带 `type: 'module'`，否则 Vite 开发模式下不会按 ES 模块处理。
- **fallback 路径**：Worker 创建抛错（CSP / 老浏览器）时，Hook 在内部捕获并改走 `lib/json.ts` 同步函数；这些函数仍然抛 Error，Hook 把抛出的 Error 转成 `{ok:false, error: e.message}` 形式以与 worker 路径对齐。
- **id 关联**：Hook 维护自增 id 和 `Map<id, {resolve}>`；每次 onmessage 按 id 找回调；卸载时 `worker.terminate()` 并对未完成 id 直接 resolve `{ok:false, error:'unmount'}` 防止 Promise 悬挂。
- **错误 position 解析**：`SyntaxError` 在 Chromium 与 Firefox 上 message 形如 `Unexpected token x in JSON at position 123` / `JSON.parse: unexpected character at line 1 column 5 of the JSON data`。Worker 端只做 `position N` 简单正则提取，提不到就不返回 `errorPos`，UI 不跳光标。
- **复制阈值**：8 MB 与 `BrowserRequestPage.tsx` 中 `PRETTY_MAX_BYTES` 对齐；如未来调整，搜 `8 * 1024 * 1024` 一并处理。
- **busy 串行**：UI 同一时刻只允许一个 op 在跑；Hook 内部并不阻止并发，由 UI 用 `disabled` 控制。
- **typecheck**：`pnpm typecheck` 或 `tsc -b --noEmit` 必须过；CodeMirror 类型从 `@codemirror/state`、`@codemirror/view` 导出，注意与 `vite.config.ts` 的 dedupe 配置一致，不要新增重复依赖路径。
- **注释语言**：本目录 `.ts/.tsx` 文件注释使用中文（kai-toolbox 项目约定）。

## 6. 测试与验证要点

- typecheck 通过
- dev server 启动后访问 `/tools/formatter`（或对应路由），JSON 子面板：
  - 小输入（粘贴 `{"a":1}`）格式化 / 压缩 / 转义 / 反转义功能正确
  - 错误 JSON（如 `{a:1}`）报错且输入框光标跳到错误位置
  - 30 MB 测试文件粘贴 → 格式化期间页面其它交互（切到 Nginx tab、滚动等）不冻结
  - 输出 > 8 MB 时复制按钮变灰、下载按钮可用
- Performance 录制确认 `JSON.parse` 落在 worker thread
- 检查 `browser-request` 页面仍正常（共享 `useIsDarkTheme` 抽离后）
