# 编码摘要文档

> 对应完整文档：`需求分析智能体实现-20260412-v2.md`
> 前序文档：`需求分析智能体实现-20260408-v1.md`（无 coding 文档）

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-08 | 初始版本（无 coding 文档） |
| v2 | 2026-04-12 | 适配画像 3 文件拆分：State 输入改为 Markdown、Prompt 重构、ImpactAnalysis 输出增强 |

---

## 1. 核心业务规则

- **R1（修改）**：affectedClasses 优先从 business-context.md 维度 4 定位，CodeSearchTool 用于**验证**而非发现
- **R2**：多维度搜索（功能/数据/集成），至少搜索 2 次 → **v2 改为 0-2 次验证性搜索**
- **R3**：只分析不设计，不输出方案
- **R4**：输出严格 JSON
- **R5（新增）**：禁止加载 coding-conventions.md，避免编码约定干扰需求分析
- **R6（新增）**：business-context.md 不存在时降级为 v1 纯搜索模式，标注 `degraded: true`
- **R7（新增）**：约束影响分析基于 business-context.md 维度 6，不凭空推测
- **R8（新增）**：跨服务影响分析基于 business-context.md 维度 8-9，不猜测服务关系

---

## 2. 接口契约

v2 不新增 HTTP 接口。内部 State 字段变更：

```text
// v1 输入
state.projectProfile: JSON (ProjectProfile 对象)
state.archTopology: JSON (ArchTopology 对象)

// v2 输入
state.projectProfile: String (project-profile.md Markdown 原文)
state.businessContext: String (business-context.md Markdown 原文)
```

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `c.e.l.application.devplan.node.AnalyzeNode` | 修改 | 加载逻辑改为读 2 个 Markdown 文件 |
| `c.e.l.infrastructure.devplan.agent.RequirementAnalyzerAgent` | 修改 | Prompt 重构 |
| `c.e.l.infrastructure.devplan.agent.DevPlanAgentConfig` | 修改 | REQUIREMENT_ANALYZER prompt 常量 |
| `c.e.l.domain.devplan.model.ImpactAnalysis` | 修改 | 新增 3 个 Optional 字段 |
| `c.e.l.domain.devplan.model.DevPlanState` | 修改 | archTopology → businessContext |

> `c.e.l` = `com.exceptioncoder.llm`

### 关键方法签名与职责

```text
// === AnalyzeNode 改造 ===
c.e.l.application.devplan.node.AnalyzeNode#execute(DevPlanState state): DevPlanState
  — 加载 project-profile.md + business-context.md → 注入 state → 调用 AgentRouter → 解析输出 → 写入 state.impactAnalysis

c.e.l.application.devplan.node.AnalyzeNode#loadBusinessContext(String projectPath): String
  — 读取 docs/business-context.md，不存在则返回 null（触发降级）

// === ImpactAnalysis 扩展 ===
c.e.l.domain.devplan.model.ImpactAnalysis
  — 新增字段：
    List<ConstraintImpact> constraintImpacts     // 受影响的约束（事务边界/幂等/状态守卫）
    List<CrossServiceImpact> crossServiceImpacts  // 需改动的下游服务
    List<EventImpact> eventImpacts               // 需新增/修改的事件

// === DevPlanState 改造 ===
c.e.l.domain.devplan.model.DevPlanState
  — 字段变更：
    @Deprecated ArchTopology archTopology  // 废弃
    String businessContext                 // 新增：business-context.md 原文
```

---

## 4. 数据结构

### ImpactAnalysis 新增字段

```java
// v2 新增（全部 Optional，向后兼容）
public record ConstraintImpact(
    String constraintType,  // TRANSACTION / IDEMPOTENT / AUTH / STATE_GUARD / COMPENSATION
    String description,     // 约束描述
    String affectedMethod,  // 受影响的方法全路径
    String impact           // NEED_MODIFY / NEED_NEW / NEED_REVIEW
) {}

public record CrossServiceImpact(
    String targetService,   // 目标服务注册名称
    String callType,        // SYNC_CALL / ASYNC_EVENT
    String currentInterface,// 当前接口（如有）
    String requiredChange   // 需要的变更描述
) {}

public record EventImpact(
    String eventType,       // 事件类型名
    String action,          // NEW / MODIFY / CONSUME
    String topic,           // Topic/Exchange
    String description      // 变更说明
) {}
```

### ImpactAnalysis 完整 JSON 输出结构（v2）

```json
{
  "requirementType": "ENHANCEMENT",
  "requirementSummary": "为订单系统增加退货退款能力",
  "degraded": false,
  "affectedModules": ["order-service"],
  "affectedClasses": [
    {
      "fullClassName": "com.xxx.order.service.OrderService",
      "filePath": "order-service/src/.../OrderService.java",
      "impact": "MODIFY",
      "reason": "新增 refundOrder 方法",
      "evidence": "business-context.md 维度 4 订单域能力清单"
    }
  ],
  "constraintImpacts": [
    {
      "constraintType": "STATE_GUARD",
      "description": "PAID 后不可直接 CANCELLED，退款需新增 PAID -> REFUNDING 守卫",
      "affectedMethod": "OrderService#changeStatus",
      "impact": "NEED_MODIFY"
    },
    {
      "constraintType": "TRANSACTION",
      "description": "退款操作需与订单状态变更在同一事务",
      "affectedMethod": "RefundService#processRefund",
      "impact": "NEED_NEW"
    },
    {
      "constraintType": "IDEMPOTENT",
      "description": "退款接口需要幂等控制，防止重复退款",
      "affectedMethod": "RefundController#requestRefund",
      "impact": "NEED_NEW"
    }
  ],
  "crossServiceImpacts": [
    {
      "targetService": "payment-service",
      "callType": "SYNC_CALL",
      "currentInterface": "POST /api/pay/create",
      "requiredChange": "需调用退款接口 POST /api/pay/refund"
    },
    {
      "targetService": "inventory-service",
      "callType": "SYNC_CALL",
      "currentInterface": "POST /api/inventory/lock",
      "requiredChange": "需调用库存归还接口 POST /api/inventory/release"
    }
  ],
  "eventImpacts": [
    {
      "eventType": "ORDER_REFUNDED",
      "action": "NEW",
      "topic": "order-events",
      "description": "退款完成后发布，供 notification-service 消费发送退款通知"
    }
  ],
  "dependencyChain": [
    "RefundController -> RefundService -> OrderService -> PaymentFeignClient",
    "RefundController -> RefundService -> OrderService -> InventoryFeignClient"
  ],
  "reusableComponents": [
    {
      "fullClassName": "com.xxx.order.service.OrderService#cancelOrder",
      "reusePlan": "退款流程可参考取消订单的库存释放逻辑，但状态流转不同"
    }
  ],
  "riskPoints": [
    {
      "risk": "退款与支付回调并发：用户发起退款的同时支付回调到达",
      "severity": "HIGH",
      "mitigation": "状态守卫 + 分布式锁确保状态变更原子性"
    }
  ],
  "newClassesNeeded": [
    {
      "suggestedFullClassName": "com.xxx.order.service.RefundService",
      "layer": "application",
      "purpose": "退款业务编排"
    },
    {
      "suggestedFullClassName": "com.xxx.order.controller.RefundController",
      "layer": "api",
      "purpose": "退款 API 入口"
    }
  ]
}
```

---

## 5. 重要约束与边界

- **上下文隔离**：AnalyzeNode 只注入 projectProfile + businessContext，**禁止注入 codingConventions**
- **降级兼容**：business-context.md 不存在时完全走 v1 路径，不报错
- **新字段向后兼容**：ImpactAnalysis 新增字段全部 Optional/nullable，DesignNode 按需消费
- **证据标注**：Agent 输出的 affectedClasses 中每条必须有 evidence 字段，标注信息来源（文档维度 or CodeSearchTool 结果）

---

## 6. 下游依赖调用

```text
// 文件读取
c.e.l.infrastructure.devplan.profile.ProfileMarkdownReader#readFull(Path): String
  → 读取 business-context.md

// Tool 调用（不变）
c.e.l.infrastructure.devplan.tool.CodeSearchTool#search(String query): List<SearchResult>
c.e.l.infrastructure.devplan.tool.FileReadTool#read(String path): String
```

---

## 7. 异常处理要点

| 场景 | 处理 |
|------|------|
| business-context.md 不存在 | `loadBusinessContext` 返回 null → 降级 v1 → 标注 `degraded: true` |
| business-context.md 格式异常 | 同上降级 |
| CodeSearchTool 不可用 | 纯文档分析，标注 `ragDegraded: true` |
| Agent 输出非法 JSON | 重试 1 次，仍失败标记 FAILED |
| ImpactAnalysis 新字段缺失 | DesignNode 容忍 null，不报错 |

---

## 8. System Prompt（v2 核心变更）

```text
你是需求分析专家，精通 DDD-lite 分层架构和 Java/Spring Boot 技术栈。

## 角色约束
- 你只负责分析需求的影响范围，不要生成设计方案
- 你必须优先从业务上下文文档中查找信息，CodeSearchTool 仅用于验证
- 每条 affectedClasses 必须标注 evidence（信息来源）

## 项目画像
{state.projectProfile}

## 业务上下文（核心参考）
{state.businessContext}

## 用户需求
{state.requirement}

## 你可以使用的工具
1. CodeSearchTool — 语义搜索项目代码，**仅用于验证文档中提到的类是否存在或查找文档未覆盖的细节**
2. FileReadTool — 读取指定文件，**仅在需要确认具体实现逻辑时使用**

## 分析框架（严格按此执行）

### Step 1: 需求分类
判断需求类型：CRUD / INTEGRATION / REFACTOR / NEW_DOMAIN / ENHANCEMENT / CROSS_CUTTING

### Step 2: 业务上下文查询
从业务上下文文档中定位：
1. **维度 3（数据模型与状态机）**：需求涉及哪些实体？需要新增/修改哪些状态？
2. **维度 4（业务能力清单）**：已有哪些相关能力可以复用？
3. **维度 5（核心业务流程）**：需求应该插入到哪个流程之后？

### Step 3: 搜索验证（按需）
对 Step 2 中定位到的关键类，用 CodeSearchTool 验证其是否存在、是否有更新。
如果文档信息已足够明确，可以跳过此步。

### Step 4: 约束与跨服务分析
从业务上下文文档中定位：
1. **维度 6（关键约束）**：哪些事务边界/幂等点/状态守卫会受影响？
2. **维度 8（对外调用服务）**：需要调用哪些下游服务的新接口？
3. **维度 9（事件契约）**：需要新增/修改哪些事件？

### Step 5: 综合输出
生成 ImpactAnalysis JSON，确保包含：
- affectedClasses（每条附 evidence）
- constraintImpacts（从维度 6 推导）
- crossServiceImpacts（从维度 8 推导）
- eventImpacts（从维度 9 推导）

## 输出格式（严格 JSON，schema 见 coding 文档第 4 节）
```

---

## 9. 实施顺序

```text
Phase 1: ImpactAnalysis 模型扩展（新增 3 个 record + ImpactAnalysis 新字段）
Phase 2: DevPlanState 字段变更（archTopology → businessContext）
Phase 3: AnalyzeNode 改造（加载 business-context.md，降级逻辑）
Phase 4: System Prompt 重构（DevPlanAgentConfig 常量更新）
Phase 5: 集成测试（退货退款场景验证）
```
