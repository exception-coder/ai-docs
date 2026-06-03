# 岗位技能趋势聚合 — 编码摘要

> 由 `岗位技能趋势聚合-current.md` 精简，供编码使用。字段级接口契约见 `岗位技能趋势聚合-api-current.md`。
> **最后更新**：2026-06-03

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-06-03 | 初始版本（对应设计 v2：4 决策已确认） |

---

## 1. 核心业务规则

- R1 JD 仅经现有入库接口推入，不新建采集
- R2 入库归一：查 `job_family_alias` 命中归标准类；未命中调 LLM 判定 + 回写映射（幂等）
- R3 归一降级：归一 LLM 失败用原始文本入库，`normalized=false`，不阻断
- R4 `job_posting_meta.raw_text` 存完整 JD 原文，抽取以原文为准
- R5 LLM 抽取输出归一化技能词（同义合并、统一大小写）
- R6 级别由 LLM 推断到 SeniorityLevel 四档，无法判定归 UNKNOWN 桶
- R7 聚合每 10 条 JD 一批调 LLM，批内单条失败跳过计数
- R8 reliable 判定：当日占比 ≥ 阈值(默认 0.30) 且 样本量 ≥ 最小数(默认 5)
- R9 快照按 (family, level, snapshot_date) 幂等覆盖
- R10 查询只读快照，不触发 LLM
- R11 LLM 并发受 Provider RateLimiter 约束，429 走既有降级

---

## 2. 接口入口指针

| 接口 | 实现类 #方法 |
|------|-------------|
| `POST /api/v1/skill-trend/aggregate` | `...api.controller.skilltrend.SkillTrendController#aggregate` |
| `GET /api/v1/skill-trend/daily` | `...api.controller.skilltrend.SkillTrendController#daily` |
| `GET /api/v1/skill-trend/trend` | `...api.controller.skilltrend.SkillTrendController#trend` |
| `POST /api/job-search/store`（增强） | `...api.controller.jobsearch.JobSearchController#storeJob`（DTO 增 rawText，归一在 JobVectorService） |

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.llm.domain.skilltrend.model.JobPostingMeta` | 新建 | JD 元数据领域模型 |
| `com.exceptioncoder.llm.domain.skilltrend.model.SkillExtractionResult` | 新建 | 单条 JD 抽取产物 |
| `com.exceptioncoder.llm.domain.skilltrend.model.SkillTrendSnapshot` | 新建 | 快照聚合根 |
| `com.exceptioncoder.llm.domain.skilltrend.model.SkillStat` | 新建 | 单技能统计项 |
| `com.exceptioncoder.llm.domain.skilltrend.repository.JobPostingMetaRepository` | 新建 | 按 family+date 枚举 |
| `com.exceptioncoder.llm.domain.skilltrend.repository.JobFamilyAliasRepository` | 新建 | 别名映射 CRUD |
| `com.exceptioncoder.llm.domain.skilltrend.repository.SkillTrendSnapshotRepository` | 新建 | 快照 upsert/查询 |
| `com.exceptioncoder.llm.domain.skilltrend.service.SkillExtractionPromptProvider` | 新建 | 抽取+归一 Prompt SPI |
| `com.exceptioncoder.llm.application.skilltrend.JobFamilyNormalizer` | 新建 | 岗位族归一 |
| `com.exceptioncoder.llm.application.skilltrend.SkillTrendAggregationService` | 新建 | 批量抽取+聚合 |
| `com.exceptioncoder.llm.application.skilltrend.SkillTrendQueryService` | 新建 | 查询快照/趋势 |
| `com.exceptioncoder.llm.application.skilltrend.SkillAggregationExecutor` | 新建 | 实现 domain SPI LlmTaskExecutor，@LlmTaskExecutorType("SKILL_AGGREGATION")；放 application（infra 不可依赖 application） |
| `com.exceptioncoder.llm.application.skilltrend.SkillTrendDailyScheduler` | 新建 | 每日定时触发（@Value skilltrend.schedule-enabled 守卫，默认关闭） |
| `com.exceptioncoder.llm.application.service.JobVectorService` | 修改 | 入库增挂归一+写 meta |
| `com.exceptioncoder.llm.infrastructure.skilltrend.SkillExtractionPromptProviderImpl` | 新建 | 两套 SystemPrompt + 默认标准岗位族清单 |
| `com.exceptioncoder.llm.infrastructure.skilltrend.persistence.*RepositoryImpl` ×3 + `*JpaRepository` ×4 + `entity.*Entity` ×4 | 新建 | JPA 实现 |
| `com.exceptioncoder.llm.api.controller.skilltrend.SkillTrendController` | 新建 | 3 端点 |
| `com.exceptioncoder.llm.api.dto.JobPostingDTO` | 修改 | 增 rawText |

### 关键方法签名与职责

```
// 归一
application.skilltrend.JobFamilyNormalizer#normalize(String raw): String
    — 查别名映射命中即返回标准族；未命中调 LLM 判定 + 回写 alias，失败返回原文(标记由调用方处理)

// 入库增强
application.service.JobVectorService#storeJobPosting(JobPosting jp): void
    — 原逻辑不变基础上：normalize(jobFamily) → 写 job_posting_meta(含 rawText/postDate) → upsert Qdrant
application.service.JobVectorService#batchStoreJobPostings(List<JobPosting>): void
    — 批量版本，逐条归一+落 meta

// 聚合
application.skilltrend.SkillTrendAggregationService#aggregate(String family, LocalDate date): SkillTrendSnapshot... 
    — 枚举 meta → 按 10 条分批 extractBatch → 按 (family,level) 聚合 → upsert 快照
application.skilltrend.SkillTrendAggregationService#extractBatch(List<JobPostingMeta>): List<SkillExtractionResult>
    — 拼 10 条进一次 prompt 调 LLMOrchestrationService，解析 JSON 数组对齐输入顺序，单条失败跳过

// 查询
application.skilltrend.SkillTrendQueryService#queryDaily(String family, SeniorityLevel level, LocalDate date)
application.skilltrend.SkillTrendQueryService#queryTrend(String family, SeniorityLevel level, String skill, LocalDate from, LocalDate to)

// 异步
infrastructure.skilltrend.task.SkillAggregationExecutor#execute(task) — 调 AggregationService.aggregate，进度经任务框架 SSE
```

---

## 4. 数据结构

### 关键表

```
表 job_posting_meta
  id BIGINT PK
  posting_id BIGINT           -- 对应 Qdrant 中的 JD
  job_family VARCHAR          -- 归一后的标准岗位族
  raw_family VARCHAR          -- 原始自由填文本
  normalized TINYINT(1)       -- 是否成功归一
  level VARCHAR               -- 推断/原始级别(可空)
  post_date DATE              -- 用于按日枚举
  raw_text TEXT               -- 完整 JD 原文
  created_at DATETIME
  INDEX idx_family_date (job_family, post_date)

表 job_family_alias
  id BIGINT PK
  alias VARCHAR UNIQUE        -- 原始写法
  standard_family VARCHAR     -- 标准岗位族
  source VARCHAR              -- LLM / MANUAL
  created_at DATETIME

表 skill_trend_snapshot
  id BIGINT PK
  job_family VARCHAR
  level VARCHAR               -- SeniorityLevel 或 UNKNOWN
  snapshot_date DATE
  jd_sample_size INT
  created_at DATETIME
  UNIQUE uk_family_level_date (job_family, level, snapshot_date)

表 skill_trend_item
  id BIGINT PK
  snapshot_id BIGINT FK
  skill_name VARCHAR
  frequency INT
  ratio DECIMAL(5,4)
  reliable TINYINT(1)
  rank INT
  INDEX idx_skill (skill_name)
```

### 关键 DTO 字段

```java
// JobPostingDTO 新增
String rawText;   // JD 原文，可选，向前兼容
```

---

## 5. 重要约束与边界

- 幂等键：聚合 = `(job_family, level, snapshot_date)` 覆盖；别名 = `alias` 唯一键
- 事务范围：快照 upsert（snapshot + items）同事务；meta 单条写入独立事务
- 阈值可配：`skilltrend.reliable-ratio`(0.30)、`skilltrend.min-sample`(5)、`skilltrend.batch-size`(10)
- 不处理：JD 去重（依赖既有 dupGroupId）；meta 表上线前历史 JD 需回填脚本另处理
- 标准岗位族候选集：归一 prompt 需注入一份初始标准分类清单（配置项）

---

## 6. 下游依赖调用

```
application.service.LLMOrchestrationService#chat(LLMRequest): LLMResponse   — 归一判定 + 批量抽取
llm-task-management 提交/执行 API（具体见该组件 coding 文档）               — 异步聚合
domain.repository.VectorStoreRepository（经 JobVectorService）              — Qdrant upsert（不变）
```

---

## 7. 异常处理要点

- 归一 LLM 失败 → 用 raw_family 入库，normalized=false，不抛
- 批量抽取整批失败 → 该批降级为逐条抽取；逐条仍失败 → 跳过并计入失败数，不中断聚合
- 当日无 meta 记录 → 不生成空快照，任务状态 SKIPPED
- 级别无法推断 → 归 UNKNOWN 桶，不丢弃
- 429 / Provider 限速 → 走既有 LLMProviderRouter 降级链
- 查询无快照 → 返回空结果（200），不报错
```
