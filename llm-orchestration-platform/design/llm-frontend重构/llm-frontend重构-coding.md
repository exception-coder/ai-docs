# llm-frontend 重构 · 编码摘要

> 配套设计文档：`llm-frontend重构-current.md` · 最后更新 2026-05-25

## 1. 核心业务规则
- R1 主题仅 light + dark；R2 sidebar w-60/w-16, topbar h-14；R3 删除 Bell/Search/假状态/假 avatar
- R4 路由顶层不再用 AnimatePresence；R5 `src/views/` 文件名 + 路由 path 不动
- R6 localStorage `app-theme` 兜底：非 'dark' 视为 light
- R7 Phase-1 视觉不破：`@layer components` alias 兜底
- R9 业务依赖保留（@xyflow/react / framer-motion / marked / highlight.js / html2canvas）

## 2. 接口入口（前端无后端接口契约变更）
本次重构不涉及 API 层；调用后端的 `src/api/*` 不动。

## 3. 涉及类清单（路径 = `llm-orchestration-platform/llm-frontend/`）

### 3.1 新建
| 路径 | 职责 |
|---|---|
| `src/lib/utils.ts` | `cn(...inputs)` = twMerge(clsx(inputs))，从 kai-toolbox 直接抄 |
| `src/components/ui/button.tsx` | shadcn-style Button + cva variants（default/destructive/outline/secondary/ghost/link）+ size（default/sm/lg/icon）+ asChild |
| `src/components/ui/card.tsx` | Card / CardHeader / CardTitle / CardDescription / CardContent / CardFooter |
| `src/components/ui/input.tsx` | shadcn-style Input forwardRef |
| `src/components/ui/badge.tsx` | Badge + cva variants（default/secondary/destructive/outline/success） |
| `src/components/ui/separator.tsx` | Separator orientation=horizontal\|vertical |
| `src/components/ui/sheet.tsx` | radix-ui Dialog 包装的 Sheet（top/bottom/left/right） |
| `src/shell/nav.ts` | menuGroups（4 组 14 项，icon + name + path）+ titleByPath（仅 title） |
| `src/shell/ThemeToggle.tsx` | useState dark，useEffect 同步 `document.documentElement.classList`，启动从 localStorage 读，写回 `'light'\|'dark'` |
| `src/shell/Sidebar.tsx` | 桌面常驻 + 移动 Sheet；分组 + 单层 NavLink；折叠 w-16/展开 w-60 |
| `src/shell/TopBar.tsx` | h-14；显示当前 path 对应 title + 折叠按钮 + 汉堡 + ThemeToggle |
| `src/shell/AppShell.tsx` | 布局壳：Sidebar + Sheet + TopBar + `<Outlet/>`；同步 `visualViewport.height` 到 `--app-vh` |
| `src/index.css` | 单一 oklch token 主题 + `.dark` 覆盖 + Phase-1 alias 兜底层（`.app-shell .app-sidebar .app-header .app-surface .app-recess .neo-convex .neo-concave .app-chip .app-btn-primary .app-input`）+ scrollbar |

### 3.2 重写
| 路径 | 改动 |
|---|---|
| `src/App.tsx` | 由 ~260 行（含 Sidebar/Header/ThemePicker）→ ~30 行：BrowserRouter + Routes + Route(element=AppShell) + 14 条子 Route + Placeholder |
| `src/main.tsx` | `import '@/styles/index.css'` → `import './index.css'` |
| `package.json` | + `class-variance-authority ^0.7.1`, + `@radix-ui/react-dialog ^1.1.x`, + `@radix-ui/react-slot ^1.1.x` |

### 3.3 Phase-3 删除
- `src/styles/` 整目录
- `src/app/navigation.ts`（内容已并入 `src/shell/nav.ts`）
- `src/index.css` 内 Phase-1 alias 兜底层

## 4. 数据结构
无数据库变更。LocalStorage 键 `app-theme` 值域：`'light' | 'dark'`（旧值 'ai-clean' / 'cyber' 等启动时被 ThemeToggle 兜底为 light）。

## 5. 重要约束与边界
- 14 个 view 文件名、路由 path、import 路径 `@/views/Xxx` 全部保留，否则会破。
- Phase-1 完成后必须 dev server 跑一遍 14 个页面 smoke test，靠 alias 兜底视觉。
- 删除 App.tsx 顶层 `AnimatePresence + motion.div`，但 view 内 framer-motion 调用全部保留。
- 不重命名既有 `src/api/ src/hooks/ src/composables/ src/utils/ src/router/ src/components/graph/` 任何文件。
- vite alias `@` → `src/` 已配置，新文件用 `@/` 引用即可。
