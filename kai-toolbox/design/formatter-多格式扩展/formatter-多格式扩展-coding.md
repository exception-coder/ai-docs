# formatter 多格式扩展 · 编码摘要

> 最后更新：2026-05-22
> 配套设计文档：`formatter-多格式扩展-current.md`

## 1. 核心业务规则（实现约束）

- 所有 lib 函数 **不抛异常**，统一返回 `{ ok: true, output } | { ok: false, error, errorPos? }`。
- Panel 渲染规则：复用 JsonPanel 的字节阈值常量（HIGHLIGHT_MAX_BYTES=1MB，COPY_MAX_BYTES=8MB，SOFT_WARN_BYTES=32MB）。
- 输入 / 输出 CodeEditor 均通过 ref 写值，不绑到 React state（避免巨型字符串触发 re-render）。
- localStorage key：`formatter.json.splitRatio`、`formatter.xml.splitRatio`、`formatter.yaml.splitRatio`、`formatter.sql.splitRatio`，互相独立。
- 仅在 `lg` 横向布局展示拖拽手柄；窄屏走 flex-col 上下堆叠。

## 2. 涉及类清单（全路径）

### 2.1 重命名

| 旧 | 新 | 变化 |
|----|----|------|
| `kai-toolbox/frontend/src/features/formatter/components/JsonEditor.tsx` | `.../components/CodeEditor.tsx` | 重命名 + 加 `language?: 'json'\|'xml'\|'yaml'\|'sql'` prop |

### 2.2 新增

| 文件 | 主要导出 / 方法签名 | 职责 |
|------|------|------|
| `frontend/src/features/formatter/lib/useSplitRatio.ts` | `useSplitRatio(storageKey, initial?, range?) → { ratio, containerRef, onSplitterPointerDown, reset }` | 抽取 JsonPanel 已实现的拖拽逻辑 |
| `frontend/src/features/formatter/lib/xml.ts` | `xmlFormat / xmlMinify / xmlEscape / xmlUnescape` | XML 4 个动作；底层 `xml-formatter` |
| `frontend/src/features/formatter/lib/yaml.ts` | `yamlFormat / yamlMinify / yamlToJson / jsonToYaml` | YAML 4 个动作；底层 `js-yaml` |
| `frontend/src/features/formatter/lib/sql.ts` | `sqlFormat / sqlMinify`；类型 `SqlDialect` / `SqlKeywordCase` | SQL 2 个动作；底层 `sql-formatter` |
| `frontend/src/features/formatter/components/SplitPane.tsx` | `<SplitPane ratio left right containerRef onSplitterPointerDown />` | 横向左右分栏 + 中间拖拽手柄的容器组件 |
| `frontend/src/features/formatter/components/XmlPanel.tsx` | `export function XmlPanel()` | Tab：缩进 + Format/Minify/Escape/Unescape |
| `frontend/src/features/formatter/components/YamlPanel.tsx` | `export function YamlPanel()` | Tab：缩进 + Format/Minify/To JSON/From JSON |
| `frontend/src/features/formatter/components/SqlPanel.tsx` | `export function SqlPanel()` | Tab：方言 + 缩进 + 关键字大小写 + Format/Minify |

### 2.3 修改

| 文件 | 变更点 |
|------|------|
| `frontend/src/features/formatter/components/JsonPanel.tsx` | 1) `JsonEditor` 引用改 `CodeEditor`；2) 内联的 leftRatio / onSplitterPointerDown 替换为 `useSplitRatio('formatter.json.splitRatio')`；3) 给 CodeEditor 传 `language="json"` |
| `frontend/src/features/formatter/pages/FormatterPage.tsx` | Tab 从 `json \| nginx` 扩展到 `json \| xml \| yaml \| sql \| nginx`；每个 Tab 配 hint 文案；按 Tab 渲染对应 Panel |
| `frontend/package.json` | 新增 `xml-formatter / js-yaml / sql-formatter / @codemirror/lang-yaml / @codemirror/lang-sql`，devDep 加 `@types/js-yaml` |

## 3. 关键操作步骤（lib 实现要点）

### 3.1 lib/xml.ts

```ts
import xmlFormatter from 'xml-formatter'
type Result = { ok: true; output: string } | { ok: false; error: string }

export function xmlFormat(input: string, opts: { indent: number | '\t' }): Result {
  try {
    const indentation = opts.indent === '\t' ? '\t' : ' '.repeat(opts.indent as number)
    return { ok: true, output: xmlFormatter(input, { indentation, collapseContent: true, lineSeparator: '\n' }) }
  } catch (e) {
    return { ok: false, error: e instanceof Error ? e.message : String(e) }
  }
}

export function xmlMinify(input: string): Result {
  try {
    // xml-formatter 不直接支持 minify；做法：先 format 再 collapse 所有换行 + 空白（仅在标签之间）
    const out = xmlFormatter(input, { indentation: '', lineSeparator: '', collapseContent: true })
    return { ok: true, output: out }
  } catch (e) {
    return { ok: false, error: e instanceof Error ? e.message : String(e) }
  }
}

const ENTITIES = { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&apos;' }
export function xmlEscape(s: string): string { return s.replace(/[&<>"']/g, c => ENTITIES[c as keyof typeof ENTITIES]) }
export function xmlUnescape(s: string): string {
  return s.replace(/&(amp|lt|gt|quot|apos);/g, (_, name) => ({ amp:'&', lt:'<', gt:'>', quot:'"', apos:"'" }[name as 'amp']!))
}
```

### 3.2 lib/yaml.ts

```ts
import yaml from 'js-yaml'
export function yamlFormat(input, opts: { indent: number }) {
  try {
    const data = yaml.load(input, { schema: yaml.JSON_SCHEMA }) // 用 JSON_SCHEMA 防止时间戳类怪类型
    return { ok: true, output: yaml.dump(data, { indent: opts.indent, noRefs: true, lineWidth: 120 }) }
  } catch (e) {
    return mapErr(e)
  }
}
export function yamlMinify(input) {
  try {
    const data = yaml.load(input, { schema: yaml.JSON_SCHEMA })
    return { ok: true, output: yaml.dump(data, { flowLevel: 0, noRefs: true, lineWidth: -1 }) }
  } catch (e) { return mapErr(e) }
}
export function yamlToJson(input, opts: { indent: number }) {
  try {
    const data = yaml.load(input, { schema: yaml.JSON_SCHEMA })
    return { ok: true, output: JSON.stringify(data, null, opts.indent) }
  } catch (e) { return mapErr(e) }
}
export function jsonToYaml(input, opts: { indent: number }) {
  try {
    const data = JSON.parse(input)
    return { ok: true, output: yaml.dump(data, { indent: opts.indent, noRefs: true, lineWidth: 120 }) }
  } catch (e) { return mapErr(e) }
}

function mapErr(e: unknown) {
  if (e instanceof yaml.YAMLException) return { ok: false as const, error: e.reason || e.message, errorPos: e.mark?.position }
  return { ok: false as const, error: e instanceof Error ? e.message : String(e) }
}
```

### 3.3 lib/sql.ts

```ts
import { format } from 'sql-formatter'
export type SqlDialect = 'sql' | 'mysql' | 'postgresql' | 'sqlite' | 'tsql'
export type SqlKeywordCase = 'upper' | 'lower' | 'preserve'

export function sqlFormat(input, opts: { indent: number | '\t'; dialect: SqlDialect; keywordCase: SqlKeywordCase }) {
  try {
    const useTabs = opts.indent === '\t'
    const tabWidth = useTabs ? 1 : (opts.indent as number)
    const out = format(input, { language: opts.dialect, tabWidth, useTabs, keywordCase: opts.keywordCase, linesBetweenQueries: 2 })
    return { ok: true, output: out }
  } catch (e) {
    return { ok: false, error: e instanceof Error ? e.message : String(e) }
  }
}

export function sqlMinify(input: string) {
  // 简单实现：剥离 -- 注释 + /* */ 注释；多空白塌缩成单空格。保留单/双引号字符串字面量。
  // 实现细节略，详见 lib/sql.ts。
}
```

### 3.4 useSplitRatio.ts

```ts
export function useSplitRatio(storageKey: string, initial = 0.5, range: [number, number] = [0.2, 0.8]) {
  const containerRef = useRef<HTMLDivElement>(null)
  const [ratio, setRatio] = useState<number>(() => readLs(storageKey, initial, range))
  useEffect(() => writeLs(storageKey, ratio), [ratio, storageKey])
  const onSplitterPointerDown = useCallback(/* pointer capture + clamp 逻辑 */, [])
  const reset = useCallback(() => setRatio(initial), [initial])
  return { ratio, containerRef, onSplitterPointerDown, reset }
}
```

## 4. 数据结构 / DDL

无数据库改动。

## 5. 重要约束与边界

- 所有 Panel 内不持有 input/output 文本到 React state，统一通过 CodeEditor ref。
- lib 返回 `Result<string>` 风格，**不**在 lib 内做 UI 副作用（不读 localStorage、不调 toast、不 console.error）。
- SQL minify 自己实现，不依赖 sql-formatter（v15 不提供 minify）；要保留字符串字面量原样，不裁剪。
- YAML 互转**会丢注释 + 锚点引用**，需要在 YAML Tab 顶部 hint 文案告知。
- XML / YAML 暂不接 Web Worker；超大文件超过软警告阈值时仅关闭高亮。

## 6. 风险与验证要点

| 项 | 验证手段 |
|----|----------|
| 5 个 Tab 切换不丢宽度记忆 | 手动切换 + 刷新页面，确认各 Tab 分栏比例独立持久化 |
| XML 错误时不崩 | 喂残缺 XML，检查错误条 + 文本框光标位置 |
| YAML JSON 互转往返一致 | 给定 JSON → YAML → JSON 应与原 JSON 等价（语义） |
| SQL 5 个方言切换 | 喂 `LIMIT` / `TOP` / `LIMIT ? OFFSET ?` 等方言差异写法，确认切方言后格式化不报错 |
| 大文件 (1-5 MB) | 输入大文件后 UI 不冻结超过 1s；超阈值时关闭高亮 |

