# 会话重命名与删除 — 编码摘要

> 配套 [会话重命名与删除-current.md](会话重命名与删除-current.md) 与 [会话重命名与删除-api-current.md](会话重命名与删除-api-current.md)。字段级契约以 api 文档为准。
> 最后更新日期：2026-06-01

## 1. 核心业务规则

- 历史重命名只写别名表，不改 transcript；list 时别名优先覆盖 title。
- 历史删除 = 移到 `~/.claude/projects/_trash/<编码cwd>/`，可恢复；同步清别名。
- `list()` / `findTranscript()` 必须排除 `_trash` 目录。
- 工具会话重命名走既有 `updateTitle`（SQLite title）。
- 行内操作按钮 stopPropagation；空名校验（前后端）。

## 2. 接口入口指针（契约见 api 文档）

| 方法 + 路径 | 实现类#方法 |
|---|---|
| `PUT /api/claude-chat/sessions/{id}/title` | `ClaudeChatSessionController#rename`（新增） |
| `DELETE /api/claude-chat/history/{sid}` | `ClaudeChatHistoryController#delete`（新增）→ `SessionHistoryService#moveToTrash` |
| `PUT /api/claude-chat/history/{sid}/alias` | `ClaudeChatHistoryController#rename`（新增）→ `SessionAliasRepository#upsert` |

## 3. 涉及类清单（全路径 + 方法签名 + 职责）

### 后端

- `…claudechat.repository.SessionAliasRepository` [新增]
  - `void upsert(String sdkSessionId, String alias)`（alias 空白则 `delete`）
  - `void delete(String sdkSessionId)`
  - `Map<String,String> findAll()` — sid → alias，供 list 左连
  - JDBC，表 `claude_chat_session_alias`。
- `…claudechat.service.SessionHistoryService` [改]
  - `list(cwd)`：选目录时 `filter(d -> !d.getFileName().toString().equals("_trash"))`；解析完成后注入 `aliasRepo.findAll()`，有别名则 `new HistorySessionView(sid, cwd, alias, ...)`。
  - `void moveToTrash(String cwd, String sdkSessionId)`：`findTranscript` 定位 → `Files.move` 到 `projects/_trash/<编码cwd>/<sid>.jsonl`（`createDirectories`，重名加 `-{epochMillis}`）→ `aliasRepo.delete(sid)`。
  - `findTranscript`：同样跳过 `_trash`。
  - 构造注入 `SessionAliasRepository`。
- `…claudechat.api.ClaudeChatSessionController` [改]
  - `@PutMapping("/{id}/title") ResponseEntity<Void> rename(@PathVariable String id, @RequestBody Map<String,String> body)`：校验 title 非空 → `repo.updateTitle` → 204。
- `…claudechat.api.ClaudeChatHistoryController` [改]
  - `@DeleteMapping("/{sid}") delete(@PathVariable String sid, @RequestParam(required=false) String cwd)` → `history.moveToTrash` → 204。
  - `@PutMapping("/{sid}/alias") rename(@PathVariable String sid, @RequestBody Map<String,String> body)` → `aliasRepo.upsert` → 204。
- `resources/db/claude-chat-schema.sql` [改]
  - `CREATE TABLE IF NOT EXISTS claude_chat_session_alias (sdk_session_id TEXT PRIMARY KEY, alias TEXT NOT NULL, updated_at INTEGER NOT NULL);`

### 前端（features/claude-chat）

- `api.ts` [改]
  - `renameSession(id, title)` → `PUT /claude-chat/sessions/{id}/title`（用 `http`）。
  - `deleteHistory(sid, cwd)` → `DELETE /claude-chat/history/{sid}?cwd=`。
  - `renameHistory(sid, alias)` → `PUT /claude-chat/history/{sid}/alias`。
- `components/SessionList.tsx` [改]：行 hover 显示「重命名/删除」；重命名 = 受控 inline `<input>`（Enter 提交→`renameSession`→`invalidateQueries(KEY)`，Esc 取消）；按钮 `e.stopPropagation()`。
- `components/HistoryList.tsx` [改]：行 hover 显示「重命名/删除」；重命名→`renameHistory`、删除→`deleteHistory`，操作后 `invalidateQueries(['claude-chat-history', query])`；需引入 `useQueryClient`；按钮 `stopPropagation`。

## 4. 数据结构

- 新表 `claude_chat_session_alias(sdk_session_id PK, alias, updated_at)`。
- 回收目录 `~/.claude/projects/_trash/<编码cwd>/<sid>.jsonl`。

## 5. 重要约束与边界

- DDL 用 `IF NOT EXISTS`（SchemaInitializer 每次启动跑）。
- `Files.move` 跨同盘 `~/.claude` 内，原子移动；重名加时间戳。
- list 与 findTranscript 两处都要排除 `_trash`。
- 注释中文（kai-toolbox 约定）。

## 6. 测试 / 验证要点

- 工具会话改名→刷新保留；历史改名→显示别名；历史删除→列表消失 + 文件入 `_trash` + 别名清除。
- `_trash` 不被扫出；续跑/消息分页/实时流不受影响。
- 前端 typecheck、后端 compile 通过。
