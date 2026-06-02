# PlanReviewer Agent 实现设计

> 本文档是「代码感知智能开发方案智能体 v2」的**子任务实现设计**。
> 父文档：`整体方案设计-20260406-v2.md`
> 前序文档：`Tool层与Agent初始化器实现/Tool层与Agent初始化器实现-20260407-v2.md`
> 同级文档：`需求分析智能体实现/`、`方案生成智能体实现/`
> 聚焦范围：PlanReviewer Agent 的完整实现链路 — 传感器链 / LLM-Judge / 路由决策

## 变更记录

| 版本 | 日期       | 修改人    | 变更内容摘要 |
|------|------------|-----------|--------------|
| v1   | 2026-04-08 | zhangkai  | 初始版本：传感器链 3 层设计 + LLM-Judge Rubric + 路由决策 |

---

## 1. 技术选型决策（本 Agent 适用）

### 1.1 PlanReviewer 与其他 Agent 的本质区别

```
其他 Agent（CodeAwareness / RequirementAnalyzer / SolutionArchitect）：
  → 需要从外部获取新信息（文件系统 / 向量库）
  → 通过 Tool 获取原始数据，LLM 理解和推理
  → Tool 是必需的

PlanReviewer：
  → 所有信息已在 State 中（document + topology）
  → 不需要获取新数据，只需要"判断"和"评估"
  → 不需要 Tool，使用 Sensor（传感器）做规则检查
```

### 1.2 Tool vs Sensor 的区别

| 维度 | Tool（@Tool） | Sensor（PlanSensor） |
|------|-------------|---------------------|
| 注册方式 | ToolRegistry，ToolScanner 自动发现 | PlanSensorChain，Spring 注入排序 |
| 调用方 | Agent 在 ReAct 循环中主动调用 | ReviewNode 在 Agent 推理前自动执行 |
| 数据方向 | 获取**新的**外部数据 | 检查 State 中**已有**数据 |
| 输出类型 | 原始数据（JSON） | 检查结果（SensorResult：passed + issues） |
| 适用场景 | 读文件、搜索代码、渲染模板 | 规则校验、合规检查、质量评审 |

### 1.3 为什么不用 MCP / 向量库？

| 技术 | 是否需要 | 理由 |
|------|---------|------|
| MCP | ❌ 不需要 | 审查基于文档本身，不需要外部系统信息 |
| 向量库 | ❌ 不需要 | 审查对象（document）已在 state 中，不需要搜索 |
| Tool | ❌ 不需要 | 不需要获取新数据 |
| Prompt | ✅ 需要 | LLM-Judge 的评分 Rubric |
| State | ✅ 需要 | document + archTopology 是审查输入 |

### 1.4 二期 MCP 扩展预留

```
PlanReviewer + MCP（二期）:
  └─ sonarqube-mcp-server
      → 对比 SonarQube 规则集，增强架构合规检查
      → 场景：企业有自定义的架构规则库
```

---

## 2. 角色定位

```
┌────────────────────────────────────────────────────────────┐
│  PlanReviewer — 方案审查专家                                 │
│                                                            │
│  输入：DevPlanDocument + ArchTopology（from State）         │
│                                                            │
│  职责：                                                     │
│  ① 架构合规检查 — 分层是否正确？依赖方向是否违规？            │
│  ② 命名规范检查 — 全类名？后缀规范？                         │
│  ③ 完整性检查 — 各章节是否齐全？类设计是否完整？              │
│  ④ 质量评审 — LLM-as-Judge 综合打分                         │
│                                                            │
│  输出：ValidationResult（评分 + 问题列表）                   │
│                                                            │
│  特殊：无外部 Tool，使用内置传感器链 + LLM 推理               │
└────────────────────────────────────────────────────────────┘
```

---

## 3. 执行流程（传感器链 + LLM 推理）

```
ReviewNode.execute(state)
  → agentRouter.route(PLAN_REVIEWER, state)
    → PlanReviewerAgent 执行：

    ╔═══════════════════════════════════════════════════════════════╗
    ║  Phase 1: 计算型传感器（毫秒级，先执行）                       ║
    ║                                                               ║
    ║  Sensor ①: ArchComplianceSensor                               ║
    ║  → 输入：document 中的类清单 + archTopology                    ║
    ║  → 正则/规则检查分层合规                                       ║
    ║  → 输出：{passed: bool, issues: [...]}                        ║
    ║                                                               ║
    ║  Sensor ②: NamingConventionSensor                             ║
    ║  → 输入：document 中的类名列表                                 ║
    ║  → 正则匹配命名规范                                            ║
    ║  → 输出：{passed: bool, issues: [...]}                        ║
    ╠═══════════════════════════════════════════════════════════════╣
    ║  Phase 2: 推理型传感器（秒级，后执行）                         ║
    ║                                                               ║
    ║  Sensor ③: LlmJudgeSensor                                     ║
    ║  → 输入：完整 document + Rubric + Phase1 的 issues             ║
    ║  → LLM 按 4 维度打分（每项 0-25）                              ║
    ║  → 输出：{scores: {...}, totalScore, issues: [...]}            ║
    ╠═══════════════════════════════════════════════════════════════╣
    ║  Phase 3: 结果汇总                                             ║
    ║                                                               ║
    ║  PlanSensorChain 汇总所有 Sensor 结果：                         ║
    ║  → allIssues = merge(sensor1 + sensor2 + sensor3)              ║
    ║  → totalScore = LlmJudge.totalScore                           ║
    ║  → passed = totalScore >= 70 AND 无 CRITICAL issue             ║
    ╚═══════════════════════════════════════════════════════════════╝
```

---

## 4. 传感器链详细设计

### 4.1 PlanSensor 接口（Domain 层）

```java
package com.exceptioncoder.llm.domain.devplan.service;

/**
 * 方案验证传感器接口。
 * 计算型传感器使用正则/规则做确定性检查（毫秒级）；
 * 推理型传感器调用 LLM 做质量评审（秒级）。
 */
public interface PlanSensor {

    /** 传感器名称 */
    String name();

    /** 传感器类型 */
    SensorType type();

    /** 执行检查 */
    SensorResult check(DevPlanDocument document, ArchTopology topology);

    /** 带上下文的检查（推理型传感器接收计算型的前置结果） */
    default SensorResult checkWithContext(
            DevPlanDocument document,
            ArchTopology topology,
            List<SensorIssue> preIssues) {
        return check(document, topology);
    }

    /** 执行顺序（越小越先） */
    int order();
}
```

### 4.2 领域模型

```java
// 传感器类型
public enum SensorType {
    COMPUTATIONAL,  // 计算型：正则/规则，毫秒级
    INFERENTIAL     // 推理型：LLM 调用，秒级
}

// 问题严重级别
public enum IssueSeverity {
    CRITICAL,  // 必须修正，阻塞通过
    MAJOR,     // 应该修正，影响评分
    MINOR      // 建议修正，不影响通过
}

// 单个问题项
public record SensorIssue(
    String sensorName,      // 来源传感器
    String ruleId,          // 规则编号（如 AC-1, NC-2）
    IssueSeverity severity, // 严重级别
    String description,     // 问题描述
    String location,        // 问题位置（类名 / 章节）
    String suggestion       // 修正建议
) {}

// 传感器检查结果
public record SensorResult(
    String sensorName,
    boolean passed,
    List<SensorIssue> issues,
    Map<String, Object> metadata  // 扩展字段（如评分明细）
) {}
```

### 4.3 ArchComplianceSensor — 架构合规检查

**职责：** 基于正则/规则检查文档中类设计的架构分层合规性。

**检查规则：**

| 规则ID | 规则 | 检查方式 | 严重级别 |
|--------|------|----------|----------|
| AC-1 | 类的包路径必须以 `com.exceptioncoder.llm` 开头 | 正则匹配 | CRITICAL |
| AC-2 | api 层类只能依赖 application / domain 层 | 包路径前缀匹配 | CRITICAL |
| AC-3 | application 层类只能依赖 domain 层 | 包路径前缀匹配 | CRITICAL |
| AC-4 | domain 层类不能依赖 infrastructure / api / application 层 | 包路径前缀匹配 | CRITICAL |
| AC-5 | infrastructure 层类只能依赖 domain 层 | 包路径前缀匹配 | MAJOR |
| AC-6 | 每个 domain 接口必须有对应的 infrastructure 实现 | 类名映射 | MAJOR |
| AC-7 | Controller 不能直接调用 Repository | 调用链检查 | CRITICAL |

**实现逻辑：**

```java
@Component
public class ArchComplianceSensor implements PlanSensor {

    private static final String BASE_PACKAGE = "com.exceptioncoder.llm";
    private static final Map<String, Set<String>> ALLOWED_DEPS = Map.of(
        "api",           Set.of("application", "domain"),
        "application",   Set.of("domain"),
        "domain",        Set.of(),
        "infrastructure", Set.of("domain")
    );

    @Override
    public String name() { return "ArchCompliance"; }

    @Override
    public SensorType type() { return SensorType.COMPUTATIONAL; }

    @Override
    public int order() { return 10; }

    @Override
    public SensorResult check(DevPlanDocument document, ArchTopology topology) {
        List<SensorIssue> issues = new ArrayList<>();

        for (ClassDesignEntry clazz : document.extractClassList()) {
            String fullName = clazz.fullClassName();

            // AC-1: 包路径前缀
            if (!fullName.startsWith(BASE_PACKAGE)) {
                issues.add(new SensorIssue(
                    name(), "AC-1", IssueSeverity.CRITICAL,
                    "包路径不以 " + BASE_PACKAGE + " 开头",
                    fullName, "修正为 " + BASE_PACKAGE + ".xxx.YyyClass 格式"
                ));
            }

            // AC-2~5: 分层依赖
            String layer = extractLayer(fullName);
            if (layer != null && clazz.dependsOn() != null) {
                for (String dep : clazz.dependsOn()) {
                    String depLayer = extractLayer(dep);
                    if (depLayer != null && !ALLOWED_DEPS.get(layer).contains(depLayer)) {
                        issues.add(new SensorIssue(
                            name(), "AC-" + ruleId(layer), IssueSeverity.CRITICAL,
                            layer + " 层不允许依赖 " + depLayer + " 层",
                            fullName + " → " + dep,
                            "调整依赖方向或移动类到正确的层"
                        ));
                    }
                }
            }
        }

        // AC-6: domain 接口有对应 infrastructure 实现
        checkInterfaceImplPairs(document, issues);

        return new SensorResult(name(), issues.isEmpty(), issues, Map.of());
    }

    private String extractLayer(String fullClassName) {
        if (fullClassName.contains(".api."))            return "api";
        if (fullClassName.contains(".application."))    return "application";
        if (fullClassName.contains(".domain."))         return "domain";
        if (fullClassName.contains(".infrastructure.")) return "infrastructure";
        return null;
    }
}
```

### 4.4 NamingConventionSensor — 命名规范检查

**职责：** 检查类名的命名规范。

**检查规则：**

| 规则ID | 规则 | 检查方式 | 严重级别 |
|--------|------|----------|----------|
| NC-1 | 类名必须是全类名格式 | 正则 `^com\.\w+(\.\w+)+$` | CRITICAL |
| NC-2 | api 层类必须以 Controller 结尾 | 后缀匹配 | MAJOR |
| NC-3 | application 层用例类必须以 UseCase 结尾 | 后缀匹配 | MAJOR |
| NC-4 | domain 层接口不含 Impl 后缀 | 后缀检查 | MINOR |
| NC-5 | infrastructure 实现类含 Impl 或技术前缀（Jpa/Redis/...） | 后缀/前缀匹配 | MINOR |
| NC-6 | DTO 以 Request / Response 结尾 | 后缀匹配 | MINOR |
| NC-7 | Entity 以 Entity 结尾 | 后缀匹配 | MINOR |
| NC-8 | 类名使用 PascalCase | 正则 `[A-Z][a-zA-Z0-9]+` | MAJOR |

**实现逻辑：**

```java
@Component
public class NamingConventionSensor implements PlanSensor {

    private static final Pattern FULL_CLASS_NAME = Pattern.compile(
        "^com\\.\\w+(\\.\\w+){2,}$"
    );

    private static final Map<String, List<String>> LAYER_SUFFIXES = Map.of(
        "api",           List.of("Controller"),
        "application",   List.of("UseCase"),
        "domain",        List.of("Service", "Repository"),  // 接口
        "infrastructure", List.of("Impl", "Jpa", "Redis", "Qdrant")  // 至少含其一
    );

    @Override
    public String name() { return "NamingConvention"; }

    @Override
    public SensorType type() { return SensorType.COMPUTATIONAL; }

    @Override
    public int order() { return 20; }

    @Override
    public SensorResult check(DevPlanDocument document, ArchTopology topology) {
        List<SensorIssue> issues = new ArrayList<>();

        for (ClassDesignEntry clazz : document.extractClassList()) {
            String fullName = clazz.fullClassName();
            String simpleName = extractSimpleName(fullName);

            // NC-1: 全类名格式
            if (!FULL_CLASS_NAME.matcher(fullName).matches()) {
                issues.add(new SensorIssue(
                    name(), "NC-1", IssueSeverity.CRITICAL,
                    "不是全类名格式",
                    fullName, "使用 com.exceptioncoder.llm.xxx.YyyClass 格式"
                ));
            }

            // NC-2~7: 分层命名后缀
            String layer = extractLayer(fullName);
            if (layer != null) {
                checkLayerNaming(layer, simpleName, fullName, issues);
            }
        }

        return new SensorResult(name(), issues.isEmpty(), issues, Map.of());
    }
}
```

### 4.5 LlmJudgeSensor — LLM-as-Judge 质量评审

**职责：** 使用 LLM 对设计文档进行综合质量评审。这是唯一的推理型传感器。

**评分 Rubric Prompt：**

```
你是代码设计文档评审专家。请对以下设计文档进行质量评审。

## 评分维度（每项 0-25 分，总分 100）

### 1. 完整性（Completeness）0-25 分
- 25：所有章节齐全，类设计完整覆盖所有分层，接口有请求响应示例，数据库有完整表设计
- 20：主要章节齐全，个别细节缺失
- 15：缺少 1-2 个重要章节
- 10：多个重要章节缺失
- 5：只有骨架，缺乏实质内容

### 2. 一致性（Consistency）0-25 分
- 25：类清单与接口设计完全对应，所有全类名一致，DTO 字段与数据库字段对应
- 20：基本一致，个别不一致
- 15：存在 2-3 处明显不一致
- 10：多处不一致
- 5：各章节相互矛盾

### 3. 可行性（Feasibility）0-25 分
- 25：与现有代码兼容，复用现有组件，不过度设计，技术方案成熟
- 20：基本可行，有小的改进空间
- 15：可行但有明显过度设计或遗漏
- 10：部分不可行
- 5：方案不切实际

### 4. 规范性（Compliance）0-25 分
- 25：全类名格式正确，命名规范 100%，分层依赖无违规
- 20：1-2 处轻微违规
- 15：3-5 处违规
- 10：多处违规
- 5：未遵循规范

## 已知架构拓扑（对照检查）
{archTopology}

## 计算型传感器已发现的问题
（这些问题已被确认，你不需要重复检查，但可以在规范性维度的评分中扣分）
{computationalSensorIssues}

## 待审查的设计文档
{document}

## 输出格式（严格 JSON）
```json
{
  "completeness": {"score": 0, "issues": ["缺失章节或内容"]},
  "consistency": {"score": 0, "issues": ["不一致的地方"]},
  "feasibility": {"score": 0, "issues": ["不可行或过度设计的点"]},
  "compliance": {"score": 0, "issues": ["规范违规项"]},
  "totalScore": 0,
  "summary": "一段话总评",
  "criticalIssues": ["必须修正的严重问题"],
  "suggestions": ["建议改进但非必须的项"]
}
```
```

**关键设计点：**

| 设计点 | 说明 |
|--------|------|
| 计算型结果注入 | Phase 1 的 issues 注入 LlmJudge Prompt，避免重复检查 |
| LLM 聚焦推理 | LLM 聚焦"一致性"和"可行性"这类需要理解力的维度 |
| temperature = 0.1 | 极低温度确保评审结果一致性（同文档多次评审得分相近） |
| 模型选择 | qwen-plus，不需要最强模型，但需要足够的理解力 |

**实现核心逻辑：**

```java
@Component
public class LlmJudgeSensor implements PlanSensor {

    private final LLMProvider llmProvider;

    @Override
    public String name() { return "LlmJudge"; }

    @Override
    public SensorType type() { return SensorType.INFERENTIAL; }

    @Override
    public int order() { return 100; } // 最后执行

    @Override
    public SensorResult checkWithContext(
            DevPlanDocument document,
            ArchTopology topology,
            List<SensorIssue> preIssues) {

        // 构造 Prompt（注入 Rubric + 文档 + 前置 issues）
        String prompt = buildJudgePrompt(document, topology, preIssues);

        // 调用 LLM
        LLMRequest request = LLMRequest.builder()
            .model("qwen-plus")
            .temperature(0.1)
            .maxTokens(2048)
            .systemPrompt(JUDGE_RUBRIC_PROMPT)
            .userMessage(prompt)
            .build();

        String response = llmProvider.chat(request).content();

        // 解析 JSON 结果
        JudgeResult judge = JSON.parse(response, JudgeResult.class);

        // 转换为 SensorResult
        List<SensorIssue> issues = new ArrayList<>();
        issues.addAll(toIssues(judge.criticalIssues(), IssueSeverity.CRITICAL));
        issues.addAll(toIssues(judge.suggestions(), IssueSeverity.MINOR));

        return new SensorResult(
            name(),
            judge.totalScore() >= 70 && judge.criticalIssues().isEmpty(),
            issues,
            Map.of(
                "scores", judge.scores(),
                "totalScore", judge.totalScore(),
                "summary", judge.summary()
            )
        );
    }
}
```

### 4.6 PlanSensorChain — 传感器链编排

```java
@Component
public class PlanSensorChain {

    private final List<PlanSensor> sensors; // Spring 按 @Order 注入

    /**
     * 执行完整的传感器链。
     * Phase 1: 计算型传感器（并行执行，毫秒级）
     * Phase 2: 推理型传感器（串行执行，秒级，接收 Phase 1 结果）
     * Phase 3: 结果汇总
     */
    public ValidationResult validate(DevPlanDocument document, ArchTopology topology) {

        // Phase 1: 计算型传感器
        List<SensorResult> computationalResults = sensors.stream()
            .filter(s -> s.type() == SensorType.COMPUTATIONAL)
            .sorted(Comparator.comparingInt(PlanSensor::order))
            .map(s -> s.check(document, topology))
            .toList();

        // 汇总 Phase 1 的 issues
        List<SensorIssue> preIssues = computationalResults.stream()
            .flatMap(r -> r.issues().stream())
            .toList();

        // Phase 2: 推理型传感器（注入 Phase 1 结果）
        List<SensorResult> inferentialResults = sensors.stream()
            .filter(s -> s.type() == SensorType.INFERENTIAL)
            .sorted(Comparator.comparingInt(PlanSensor::order))
            .map(s -> s.checkWithContext(document, topology, preIssues))
            .toList();

        // Phase 3: 汇总
        return aggregate(computationalResults, inferentialResults);
    }

    private ValidationResult aggregate(
            List<SensorResult> computational,
            List<SensorResult> inferential) {

        List<SensorIssue> allIssues = new ArrayList<>();
        computational.forEach(r -> allIssues.addAll(r.issues()));
        inferential.forEach(r -> allIssues.addAll(r.issues()));

        // 从 LlmJudge 获取总分
        int totalScore = inferential.stream()
            .filter(r -> r.sensorName().equals("LlmJudge"))
            .findFirst()
            .map(r -> (int) r.metadata().get("totalScore"))
            .orElse(0);

        // 计算型 CRITICAL issues 额外扣分
        long criticalCount = allIssues.stream()
            .filter(i -> i.severity() == IssueSeverity.CRITICAL)
            .count();

        boolean passed = totalScore >= 70 && criticalCount == 0;

        return new ValidationResult(passed, totalScore, allIssues);
    }
}
```

---

## 5. 路由决策（ReviewRoutingStrategy）

传感器链输出 ValidationResult 后，由 Application 层的 `ReviewRoutingStrategy` 做路由：

```java
// 位于 Application 层
public class ReviewRoutingStrategy {

    public RoutingDecision route(DevPlanState state) {
        ValidationResult result = state.validationResult();
        int count = state.correctionCount();

        if (result.passed()) {
            // 通过 → 结束
            return RoutingDecision.END;
        }

        if (count < 3) {
            // 不通过 + 修正次数未超限 → 回退到 DesignNode
            // 只将失败 issues 写入 state，供 SolutionArchitect 修正
            state.setReviewIssues(result.issues().stream()
                .filter(i -> i.severity() != IssueSeverity.MINOR)  // MINOR 不强制修正
                .toList());
            return RoutingDecision.RETRY_DESIGN;
        }

        // 修正超限 → 带问题通过
        state.setStatus("APPROVED_WITH_ISSUES");
        return RoutingDecision.END_WITH_ISSUES;
    }
}
```

**路由决策矩阵：**

| 条件 | 决策 | state 变更 |
|------|------|-----------|
| `score >= 70` 且无 CRITICAL | `END` | `status = APPROVED` |
| `score < 70` 且 `count < 3` | `RETRY_DESIGN` | `reviewIssues = [...]`, `correctionCount++` |
| 有 CRITICAL issue 且 `count < 3` | `RETRY_DESIGN` | 同上 |
| `count >= 3` | `END_WITH_ISSUES` | `status = APPROVED_WITH_ISSUES` |

**成功静默 / 失败响亮原则：**

```
✅ 通过 → 不反馈任何信息给 SolutionArchitect
   理由：成功的部分是正确的，反馈只会引入噪音

❌ 不通过 → 只反馈 CRITICAL + MAJOR issues
   理由：SolutionArchitect 只需要知道"什么地方错了"
   MINOR issues 不强制修正，不注入修正上下文
```

---

## 6. System Prompt 设计

PlanReviewer Agent 的 Prompt 较轻量，因为主要工作由传感器链完成：

```
你是方案审查协调者。你的职责是协调传感器链的审查结果，给出最终判定。

## 工作流程
1. 传感器链已自动执行并给出检查结果（你会收到传感器链的输出）
2. 你需要综合所有传感器的输出，确认结果合理性
3. 给出最终判定

## 判定规则
- totalScore >= 70 且无 CRITICAL issue → 通过
- totalScore < 70 或有 CRITICAL issue → 需修正（返回 issues 列表）
- 已修正 3 次仍不通过 → 上报（标记 approved_with_issues）

## 输出格式
直接输出 ValidationResult JSON，格式如下：
{
  "passed": true/false,
  "totalScore": 0-100,
  "issues": [...],
  "summary": "一句话总评"
}
```

---

## 7. State 交互

### 7.1 输入

| State 字段 | 类型 | 来源 |
|-----------|------|------|
| `state.document` | String (Markdown) | DesignNode |
| `state.archTopology` | JSON | ScanNode |
| `state.correctionCount` | int | 累计修正次数 |

### 7.2 输出

| State 字段 | 类型 | 说明 |
|-----------|------|------|
| `state.validationResult` | ValidationResult | 评分 + 问题列表 |
| `state.reviewIssues` | List\<SensorIssue\> | 仅失败时写入，供 DesignNode 修正 |
| `state.status` | String | APPROVED / APPROVED_WITH_ISSUES |

---

## 8. Agent 定义注册

```java
AgentDefinition.builder()
    .id("devplan-plan-reviewer")
    .name("方案审查专家")
    .description("架构合规检查、命名规范检查、LLM-Judge 综合评审")
    .systemPrompt(PLAN_REVIEWER_PROMPT)
    .toolIds(List.of())  // 无外部工具
    .modelConfig(ModelConfig.of("qwen-plus"))
    .maxTokens(4096)
    .temperature(0.1)  // 极低温度，评审需要确定性
    .build();
```

**模型选择理由：** `qwen-plus` — 审查需要理解力但不需要生成长文本。LlmJudgeSensor 内部也使用 qwen-plus。

---

## 9. 类清单

| 全类名 | 类型 | 说明 | 操作 |
|--------|------|------|------|
| **Domain 层** | | | |
| `c.e.l.domain.devplan.service.PlanSensor` | Interface | 传感器接口 | 新建 |
| `c.e.l.domain.devplan.model.SensorType` | Enum | COMPUTATIONAL / INFERENTIAL | 新建 |
| `c.e.l.domain.devplan.model.SensorResult` | Record | 传感器检查结果 | 新建 |
| `c.e.l.domain.devplan.model.SensorIssue` | Record | 单个问题项 | 新建 |
| `c.e.l.domain.devplan.model.IssueSeverity` | Enum | CRITICAL / MAJOR / MINOR | 新建 |
| `c.e.l.domain.devplan.model.ValidationResult` | Record | 最终验证结果 | 新建 |
| **Infrastructure 层 — 传感器** | | | |
| `c.e.l.infrastructure.devplan.sensor.ArchComplianceSensor` | PlanSensor 实现 | 架构合规（计算型，order=10） | 新建 |
| `c.e.l.infrastructure.devplan.sensor.NamingConventionSensor` | PlanSensor 实现 | 命名规范（计算型，order=20） | 新建 |
| `c.e.l.infrastructure.devplan.sensor.LlmJudgeSensor` | PlanSensor 实现 | LLM 质量评审（推理型，order=100） | 新建 |
| `c.e.l.infrastructure.devplan.sensor.PlanSensorChain` | Component | 传感器链编排 | 新建 |
| **Infrastructure 层 — Agent** | | | |
| `c.e.l.infrastructure.devplan.agent.PlanReviewerAgent` | Agent | 方案审查执行体 | 新建 |
| **Application 层** | | | |
| `c.e.l.application.devplan.node.ReviewNode` | Node 编排 | 调用 AgentRouter → 写入 state.validationResult | 新建 |
| `c.e.l.application.devplan.ReviewRoutingStrategy` | 路由策略 | 根据 ValidationResult 路由决策 | 新建 |

> `c.e.l` = `com.exceptioncoder.llm`

---

## 10. 核心业务规则

| # | 规则 | 说明 |
|---|------|------|
| R1 | **计算型传感器先于推理型执行** | 毫秒级先跑完，结果注入推理型 |
| R2 | **计算型结果注入 LlmJudge** | 避免 LLM 重复检查已确定的问题 |
| R3 | **score >= 70 且无 CRITICAL → 通过** | 两个条件缺一不可 |
| R4 | **修正最多 3 次** | 超过标记 approved_with_issues |
| R5 | **成功静默，失败响亮** | 只将 CRITICAL + MAJOR issues 注入修正上下文 |
| R6 | **MINOR issues 不强制修正** | 不注入修正上下文，只在最终报告中展示 |
| R7 | **LlmJudge temperature <= 0.1** | 确保评审一致性 |

---

## 11. 异常处理

| 场景 | 处理方式 |
|------|----------|
| ArchComplianceSensor 无法解析文档中的类清单 | 跳过该 Sensor，issues 为空，由 LlmJudge 兜底 |
| NamingConventionSensor 正则匹配异常 | 捕获异常，记录 WARN，跳过该规则 |
| LlmJudgeSensor LLM 调用失败 | 重试 2 次，仍失败则只采用计算型 Sensor 结果，totalScore 按 `50 - criticalCount * 10` 估算 |
| LlmJudge 返回非法 JSON | 重试 1 次，仍失败则 totalScore = 0，强制进入修正循环 |
| 文档为空 | 直接返回 `passed=false, score=0`，不执行传感器链 |

---

## 12. 测试要点

| 测试项 | 类型 | 说明 |
|--------|------|------|
| ArchComplianceSensor 检测 domain→infrastructure 违规 | 单元测试 | 构造违规类清单，验证 AC-4 触发 |
| ArchComplianceSensor 通过合法文档 | 单元测试 | 构造合规文档，验证 passed=true |
| NamingConventionSensor 检测非全类名 | 单元测试 | 输入 "UserService"，验证 NC-1 触发 |
| NamingConventionSensor 检测后缀违规 | 单元测试 | api 层类名不含 Controller |
| PlanSensorChain 执行顺序 | 单元测试 | 验证 COMPUTATIONAL 先于 INFERENTIAL |
| PlanSensorChain 计算型结果注入推理型 | 单元测试 | Mock LlmJudge，验证收到 preIssues |
| LlmJudge 评分一致性 | 集成测试 | 同文档连续评审 3 次，分差 < 10 |
| ReviewRoutingStrategy 通过决策 | 单元测试 | score=78 无 CRITICAL → END |
| ReviewRoutingStrategy 回退决策 | 单元测试 | score=55 count=1 → RETRY_DESIGN |
| ReviewRoutingStrategy 超限决策 | 单元测试 | count=3 → END_WITH_ISSUES |
| 修正循环端到端 | 集成测试 | Mock LLM 第一次低分、第二次高分，验证回退+通过 |
