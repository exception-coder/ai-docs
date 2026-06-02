# 历史会话消息加载 — 接口契约

> 配套 [历史会话消息加载-current.md](历史会话消息加载-current.md)。本期新增 1 个只读 REST 端点，无 WS 协议变更。
> 最后更新日期：2026-06-01

---

## 1. 分页读取会话历史消息

### `GET /api/claude-chat/history/{sdkSessionId}/messages`

按行游标向前分页读取某会话 transcript 的消息。

**请求**

| 项 | 说明 |
|----|------|
| Path `sdkSessionId` | SDK 会话 id（= transcript 文件名去扩展名） |
| Query `cwd` | 会话工作目录（用于定位 `~/.claude/projects/<编码cwd>/`）；可空则按 sid 跨目录找 |
| Query `before`（可选） | 行游标 = 最早已加载条目的全局索引；不传 = 取最近一页 |
| Query `limit`（可选） | 每页条数，默认 30 |

**响应 200**

```json
{
  "items": [
    { "id": "h12", "kind": "user", "text": "帮我看下这个报错" },
    { "id": "h13", "kind": "assistant", "text": "好的，我先读取日志…" },
    { "id": "h14", "kind": "tool", "toolName": "Read", "input": {"file":"a.log"}, "output": "…", "isError": false },
    { "id": "h20", "kind": "result", "stopReason": "end_turn" }
  ],
  "nextBefore": 12
}
```

| 字段 | 说明 |
|------|------|
| `items` | 本页消息，**按时间正序**（早 → 晚）；形状与前端 `ChatItem` 对齐 |
| `items[].id` | 稳定 id，历史项用 `h{全局行索引}` 前缀，与实时项 `i{seq}` 隔离 |
| `items[].kind` | `user` / `assistant` / `tool` / `result`（与实时事件渲染共用） |
| `nextBefore` | 下一页（更早）游标；本批最早条目的全局索引。**为 0 或 null 表示已到顶**，无更早 |

**消息项字段（按 kind）**

| kind | 字段 |
|------|------|
| `user` | `text` |
| `assistant` | `text` |
| `tool` | `toolName`、`input`、`output?`、`isError?` |
| `result` | `stopReason` |

**错误**

| HTTP | 场景 |
|------|------|
| 200 + 空 items | transcript 不存在 / 为空 / 全部解析失败（前端按"无历史"处理，不报错） |

> 实现：`ClaudeChatHistoryController#messages` → `SessionHistoryService#readMessages(cwd, sdkSessionId, before, limit)`。

---

## 2. 调用约定

- **首屏**：`before` 不传，取最近 `limit` 条，前端渲染后滚到底。
- **上拉翻页**：`before` = 上一次响应的 `nextBefore`，取更早 `limit` 条，前端 prepend。
- **终止**：响应 `nextBefore` 为 0/null 时，前端标记 `exhausted`，停止继续请求。
