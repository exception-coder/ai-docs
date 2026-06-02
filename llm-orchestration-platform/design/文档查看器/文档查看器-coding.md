# 编码摘要 — 文档查看器 v2

## 变更记录
- v1 → v2: 目录结构由 LLM 解析 `docs/README.md` 生成（替代文件系统扫描），解析结果持久化到数据库，支持版本管理和 diff 比对；引入 DocRefreshUseCase + DocStructureAgent（ReAct链路）

---

## 与 v1 的关键差异

| 维度 | v1 | v2 |
|---|---|---|
| 目录来源 | 文件系统扫描（物理路径） | LLM 解析 docs/README.md（语义） |
| 目录存储 | 无（每次扫描） | DB `doc_structure_version` 表 |
| 版本管理 | 无 | 有，可查看历史、可回溯 |
| 更新方式 | 硬编码流程 | Agent ReAct 链路（LLM 自主决策） |
| README 变更检测 | 无 | hash 比对，避免重复解析 |

---

## 涉及类清单

| 全类名 | 所属模块 | 职责 |
|---|---|---|
| `com.exceptioncoder.llm.api.controller.DocViewerController` | llm-api | HTTP 入口，新增 `/docs/refresh` 端点 |
| `com.exceptioncoder.llm.application.usecase.DocRefreshUseCase` | llm-application | 读取 README + 计算 hash，触发 Agent 执行 |
| `com.exceptioncoder.llm.domain.model.DocTreeNode` | llm-domain | 目录树节点（复用 v1） |
| `com.exceptioncoder.llm.domain.model.DocContent` | llm-domain | 文档内容（复用 v1） |
| `com.exceptioncoder.llm.domain.model.DocStructureVersion` | llm-domain | 版本记录模型 |
| `com.exceptioncoder.llm.domain.repository.DocStructureVersionRepository` | llm-domain | 版本仓储接口 |
| `com.exceptioncoder.llm.domain.service.DocStructureAgent` | llm-domain | Agent 接口（ReAct 循环） |
| `com.exceptioncoder.llm.infrastructure.agent.DocStructureAgentImpl` | llm-infrastructure | Agent 实现，调用 LLM |
| `com.exceptioncoder.llm.infrastructure.tool.GetLastDocStructureTool` | llm-infrastructure | @Tool：查询上一版本目录 |
| `com.exceptioncoder.llm.infrastructure.tool.SaveDocStructureTool` | llm-infrastructure | @Tool：保存新版本目录 |
| `com.exceptioncoder.llm.infrastructure.repository.DocStructureVersionRepositoryImpl` | llm-infrastructure | 版本仓储 JPA 实现 |

---

## 接口设计

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/v1/docs/tree` | 获取目录树（来自 DB） |
| GET | `/api/v1/docs/content` | 读取文档内容（文件系统） |
| GET | `/api/v1/docs/search` | AI 语义检索（Qdrant） |
| POST | `/api/v1/docs/index` | 手动触发文档索引构建 |
| POST | `/api/v1/docs/refresh` | **新增** 手动触发目录更新（Agent ReAct） |

---

## 核心业务规则

| 规则 | 说明 |
|---|---|
| 目录来源权威 | `docs/README.md` 是目录结构唯一来源，不扫描文件系统 |
| README hash 变更检测 | README 内容 hash 变化时才触发 Agent 解析 |
| 目录结构版本化 | 每次更新新建一条 `doc_structure_version` 记录 |
| 节点 path 稳定性 | Agent 解析时携带上一版本结果，保证 path 不漂移 |
| 索引 ID 格式 | 沿用 v1：`doc:{path}` |

---

## 数据库设计

### 新增表

```sql
CREATE TABLE doc_structure_version (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    version INT NOT NULL,
    readme_hash VARCHAR(64) NOT NULL,
    tree_json TEXT NOT NULL,
    diff_summary TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_version (version)
);
```
