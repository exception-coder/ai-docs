# 会话重命名与删除 — 接口契约

> 配套 [会话重命名与删除-current.md](会话重命名与删除-current.md)。本期新增 3 个端点，均在既有 `/api/claude-chat` 下。
> 最后更新日期：2026-06-01

---

## 1. 工具会话重命名

### `PUT /api/claude-chat/sessions/{id}/title`

```json
// 请求 body
{ "title": "联调退款流程" }
```

| 项 | 说明 |
|----|----|
| Path `id` | 工具会话 id（claude_chat_session.id） |
| Body `title` | 新标题，非空、trim、长度上限（如 ≤100） |

**响应**：`204 No Content`。错误：`400 BAD_TITLE`（空/超长）、`404 SESSION_NOT_FOUND`。

> 实现：`ClaudeChatSessionController#rename` → `ClaudeChatSessionRepository.updateTitle(id, title)`。

---

## 2. 本机历史删除（移回收）

### `DELETE /api/claude-chat/history/{sdkSessionId}`

| 项 | 说明 |
|----|----|
| Path `sdkSessionId` | transcript 文件名（去扩展名） |
| Query `cwd`（可选） | 定位项目目录；空则跨目录按文件名找 |

**行为**：把 `<sdkSessionId>.jsonl` 移到 `~/.claude/projects/_trash/<编码cwd>/`（重名加时间戳），并清该 sid 别名。
**响应**：`204`。文件不存在也返回 `204`（幂等）。

> 实现：`ClaudeChatHistoryController#delete` → `SessionHistoryService#moveToTrash(cwd, sid)`。

---

## 3. 本机历史重命名（别名）

### `PUT /api/claude-chat/history/{sdkSessionId}/alias`

```json
{ "alias": "Meegle MCP 接入排查" }
```

| 项 | 说明 |
|----|----|
| Path `sdkSessionId` | transcript 文件名（去扩展名） |
| Body `alias` | 自定义别名，非空 trim；传空串可视为清除别名（回落解析标题） |

**响应**：`204`。

> 实现：`ClaudeChatHistoryController#rename` → `SessionAliasRepository.upsert(sid, alias)`（空则 delete）。
> 列表 `GET /api/claude-chat/history` 的返回中，`title` 字段在有别名时即为别名（`SessionHistoryService.list` 左连别名表覆盖）。

---

## 4. 既有端点（不变）

- `GET /api/claude-chat/sessions` — 工具会话列表
- `DELETE /api/claude-chat/sessions/{id}` — 工具会话删除
- `GET /api/claude-chat/history?cwd=` — 本机历史列表（现返回的 `title` 已含别名覆盖）
- `GET /api/claude-chat/history/{sid}/messages` — 历史消息分页
