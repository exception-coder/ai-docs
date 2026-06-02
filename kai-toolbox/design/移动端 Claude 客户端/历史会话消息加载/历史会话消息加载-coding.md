# 历史会话消息加载 — 编码摘要

> 配套 [历史会话消息加载-current.md](历史会话消息加载-current.md) 与 [历史会话消息加载-api-current.md](历史会话消息加载-api-current.md)。字段级契约以 api 文档为准。
> 最后更新日期：2026-06-01

## 1. 核心业务规则

- 只读 SDK transcript jsonl，不落库。
- 行序即时间序；`before` = 最早已加载条目的全局行索引，向前翻页。
- 历史项 id 前缀 `h{index}`，与实时项 `i{seq}` 隔离，防 key 冲突。
- 历史在前、实时 append 在后，同一 items。
- prepend 更早页后用 scrollHeight 差补偿 scrollTop，不跳动；prepend 不触发滚底。
- nextBefore=0/null → exhausted，停止请求。新会话不请求历史。

## 2. 接口入口指针（契约见 api 文档）

| 方法 + 路径 | 实现类#方法 |
|---|---|
| `GET /api/claude-chat/history/{sdkSessionId}/messages` | `ClaudeChatHistoryController#messages`（新增） |

## 3. 涉及类清单（全路径 + 方法签名 + 职责）

### 后端

- `…claudechat.service.SessionHistoryService` [改]
  - `MessagePage readMessages(String cwd, String sdkSessionId, Integer before, int limit)`
    - 定位 jsonl：复用 `encode(cwd)` + `matchesProject`；cwd 空时跨目录按 `<sid>.jsonl` 找。
    - 顺序读全文件 → `List<ChatMessageView> all`（带全局索引）。
    - 分页：before==null → `all` 尾部 limit；否则 `all[max(0,before-limit) .. before)`。
    - `nextBefore` = 本批首项全局索引（>0 还有更早，0 到顶）。
  - 私有 `List<ChatMessageView> parseAll(Path jsonl)`：逐行映射——
    - user+text(非 tool_result)→ user 项；assistant text block → assistant 项；assistant tool_use → tool 项(toolName/input)；user tool_result → 回填同 tool_use_id 的 tool 项 output，否则独立 tool 项；result → result 项(stopReason)；其余跳过。
    - 复用既有 `extractText`；新增 tool_use_id 回填逻辑。
- `…claudechat.api.dto.ChatMessageView` [新增] — `record ChatMessageView(String id, String kind, String text, String toolName, Object input, String output, Boolean isError, String stopReason)`（按 kind 取用，未用字段为 null）。
- `…claudechat.api.dto.MessagePage` [新增] — `record MessagePage(List<ChatMessageView> items, Integer nextBefore)`。
- `…claudechat.api.ClaudeChatHistoryController` [改]
  - `@GetMapping("/{sdkSessionId}/messages") MessagePage messages(@PathVariable String sdkSessionId, @RequestParam(required=false) String cwd, @RequestParam(required=false) Integer before, @RequestParam(defaultValue="30") int limit)`。

### 前端（features/claude-chat）

- `api.ts` [改]：`loadMessages(sdkSessionId, cwd, before?, limit=30): Promise<{ items: ChatItem[]; nextBefore: number | null }>`（用既有 `http`）。
- `hooks/useClaudeChatSocket.ts` [改]：
  - state：`historyBefore: number | null`、`historyLoading`、`historyExhausted`。
  - `loadHistory(reset: boolean)`：reset → before=null 取最近页并设 items 为历史项；否则 before=historyBefore 取更早页 prepend。更新 historyBefore/exhausted。
  - 触发：`ready`(switch/resume) 后若是历史会话则 `loadHistory(true)`；`open` 新会话不触发。
  - 导出 `loadHistory`、`historyLoading`、`historyExhausted`。
- `components/MessageList.tsx` [改]：
  - props 加 `onLoadEarlier`、`loadingEarlier`、`exhausted`。
  - 容器加 ref + `onScroll`：`scrollTop < 80 && !loadingEarlier && !exhausted` → 记录 scrollHeight → `onLoadEarlier()`。
  - prepend 后 `useLayoutEffect` 比较 scrollHeight 差补偿 scrollTop。
  - 滚底逻辑：仅首屏/新增实时消息且原本贴底时；prepend 时跳过。
  - 顶部提示「加载更早…」/「没有更早了」。
- `pages/ChatPage.tsx` [改]：把 `onLoadEarlier={chat.loadHistory}` 等接进 `<MessageList>`。

## 4. 数据结构

- 无 DB 变更。新增传输 DTO `ChatMessageView` / `MessagePage`，前端复用 `ChatItem` 形状渲染。
- 游标 = jsonl 全局行级消息索引（解析后列表下标）。

## 5. 重要约束与边界

- 单会话全解析 + 内存切片；解析失败行跳过，整体失败回空列表不抛。
- tool_use/tool_result 按 tool_use_id 配对回填。
- 历史项 id `h{index}`；实时项 `i{seq}` 不变。
- prepend 保位用 useLayoutEffect（DOM 更新后、绘制前补偿）。
- 注释中文（kai-toolbox 约定）。

## 6. 测试 / 验证要点

- 有历史会话 switch/resume → 最近 30 条 + 贴底。
- 上拉 → 更早页 prepend、不跳动、到顶停止。
- 历史后发新消息 → 实时 append 衔接正常。
- 新会话不请求历史；空/坏 transcript 不崩。
- 回归：实时流 / 弹窗 / 附件 / 语音 / 模式切换不受影响；typecheck + compile 通过。
