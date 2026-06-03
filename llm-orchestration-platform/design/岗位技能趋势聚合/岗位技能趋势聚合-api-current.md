# 岗位技能趋势聚合 — 接口文档

> 与 `岗位技能趋势聚合-current.md` 配套，字段级契约在此维护。
> **最后更新**：2026-06-03

---

## 变更记录

| 版本 | 日期 | 修改人 | 变更内容摘要 |
|------|------|--------|--------------|
| v1 | 2026-06-03 | zhangkai | 初始版本（草稿，待确认） |

---

## 接口清单

| # | 方法 | 路径 | 用途 |
|---|------|------|------|
| 1 | POST | /api/v1/skill-trend/aggregate | 触发某岗位族某日的技能聚合（异步，返回 taskId） |
| 2 | GET | /api/v1/skill-trend/daily | 查当日（或指定日）某级别技能榜单 |
| 3 | GET | /api/v1/skill-trend/trend | 查某级别某技能的历史趋势序列 |

---

## 1. 触发聚合

### 1.1 基本信息

- **方法**：`POST`
- **路径**：`/api/v1/skill-trend/aggregate`
- **用途**：对某 jobFamily + 日期窗口的 JD 跑 LLM 抽取与聚合，落每日快照
- **认证**：不需要（内部）
- **幂等**：是，幂等键 = `jobFamily + level + snapshotDate`，重复触发覆盖当日快照

### 1.2 请求 Body

```json
{ "jobFamily": "后端开发", "snapshotDate": "2026-06-03" }
```

| 字段 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `jobFamily` | string | 是 | - | 岗位族（聚合分组维度） |
| `snapshotDate` | string(date) | 否 | 当天 | 聚合的目标日；缺省取当天 |

### 1.3 响应（200）

```json
{ "taskId": "tsk_xxx", "status": "SUBMITTED" }
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `taskId` | string | llm-task-management 任务 id，进度经其 SSE 端点订阅 |
| `status` | string | SUBMITTED / RUNNING / ... |

#### 错误

| 错误码 | HTTP | 触发场景 |
|--------|------|---------|
| `INVALID_PARAM` | 400 | jobFamily 为空 / 日期格式错 |
| `NO_JD_FOUND` | 200(空) | 当日该岗位族无 JD（不生成快照，status=SKIPPED） |

---

## 2. 查当日技能榜单

### 2.1 基本信息

- **方法**：`GET`
- **路径**：`/api/v1/skill-trend/daily`
- **用途**：返回某 `(jobFamily, level, date)` 的技能榜单
- **认证**：不需要

### 2.2 请求 Query

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `jobFamily` | string | 是 | 岗位族 |
| `level` | string | 否 | JUNIOR/INTERMEDIATE/SENIOR/EXPERT；缺省返回全部级别 |
| `date` | string(date) | 否 | 缺省取最新快照日 |

### 2.3 响应（200）

```json
{
  "jobFamily": "后端开发",
  "snapshotDate": "2026-06-03",
  "levels": [
    {
      "level": "SENIOR",
      "jdSampleSize": 42,
      "skills": [
        { "skill": "Kubernetes", "frequency": 28, "ratio": 0.67, "reliable": true, "rank": 1 },
        { "skill": "分布式事务", "frequency": 19, "ratio": 0.45, "reliable": true, "rank": 2 }
      ]
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `levels[].jdSampleSize` | integer | 该级别当日参与统计的 JD 数 |
| `skills[].ratio` | number | 出现占比 0~1 |
| `skills[].reliable` | boolean | 是否达可靠阈值（见设计文档 R4） |
| `skills[].rank` | integer | 榜单排名 |

---

## 3. 查技能历史趋势

### 3.1 基本信息

- **方法**：`GET`
- **路径**：`/api/v1/skill-trend/trend`
- **用途**：返回某 `(jobFamily, level, skill)` 在时间窗内的占比序列，用于画趋势线
- **认证**：不需要

### 3.2 请求 Query

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `jobFamily` | string | 是 | 岗位族 |
| `level` | string | 是 | 级别枚举 |
| `skill` | string | 否 | 指定技能；缺省返回 TopN 技能的多条序列 |
| `from` | string(date) | 是 | 起始日 |
| `to` | string(date) | 是 | 结束日 |

### 3.3 响应（200）

```json
{
  "jobFamily": "后端开发",
  "level": "SENIOR",
  "series": [
    { "skill": "Kubernetes", "points": [ { "date": "2026-06-01", "ratio": 0.55 }, { "date": "2026-06-03", "ratio": 0.67 } ] }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `series[].points[].ratio` | number | 该日占比；缺快照的日期不补点 |
