# 编码摘要文档

> 本文档由完整设计文档精简而来，供 AI 辅助编码时使用。
> 对应完整文档：`代码感知智能体实现-20260412-v3.md`
> 父级设计文档：`整体方案设计-20260406-v2-coding.md`
> 前序文档：`代码感知智能体实现-20260411-v2-coding.md`

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-08 | 初始版本：从 Tool 层文档抽离 |
| v2 | 2026-04-11 | 代码感知层重构：SPI 生成器链 + CLI 主实现 |
| v3 | 2026-04-12 | ClaudeCodeProfileGenerator 改造：SDK 桥接 + 空闲超时 + 双模式 |

---

## 1. 核心业务规则

保留 v2 全部规则（R1-R10），v3 新增：

- **R11**：SDK 模式为默认推荐，CLI 为备选。`mode` 配置项切换
- **R12**：空闲超时优先于总超时。有 stdout 输出就不中断
- **R13**：桥接脚本输出 JSON Lines，type 区分 progress / tool_use / result / error / final_output
- **R14**：`isAvailable()` 根据 mode 检查不同依赖（SDK: node + bridge 脚本，CLI: claude 命令）
- **R15**：`executeAndWait` 公共方法，SDK 和 CLI 共享进程监控逻辑

---

## 2. 接口契约

v3 不涉及 HTTP 接口变更。内部类方法变更见第 3 节。

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator` | 重构 | 拆为 generateViaSdk / generateViaCli / executeAndWait 三方法 + isAvailable 双分支 |
| `c.e.l.infrastructure.devplan.config.DevPlanProfileProperties.ClaudeCode` | 修改 | 新增 5 字段，去掉 timeoutSeconds |
| `scripts/claude-code-bridge.mjs` | 新建 | Node.js SDK 桥接脚本 |
| `scripts/package.json` | 新建 | npm 依赖声明 |
| `llm-starter/.../config/dev/graph.yml` | 修改 | 新增 SDK 相关配置项 |

> `c.e.l` = `com.exceptioncoder.llm`

### 关键方法签名与职责

```text
// === ClaudeCodeProfileGenerator（重构） ===

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#generate(String projectPath): Optional<String>
  — 入口：根据 config.mode 分发到 SDK 或 CLI

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#generateViaSdk(String projectPath): Optional<String>
  — SDK 模式：构建 JSON 配置，启动 node bridge 脚本，委托 executeAndWait

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#generateViaCli(String projectPath): Optional<String>
  — CLI 模式：构建 claude --print 命令，redirectInput /dev/null，委托 executeAndWait

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#executeAndWait(ProcessBuilder, String, Path, Consumer<String>): Optional<String>
  — 公共引擎：启动子进程 → 独立线程读 stdout → AtomicLong 记录 lastActivityTime → 每秒轮询空闲/总超时 → 检查 exitValue → 读取 profile.md

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#isAvailable(): boolean
  — 根据 mode 委托 isSdkAvailable() 或 isCliAvailable()

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#isSdkAvailable(): boolean
  — 检查 bridge 脚本文件存在 + node --version 可执行

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#isCliAvailable(): boolean
  — 检查 claude --version 可执行（沿用 v2 逻辑）

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#escapeJson(String): String
  — JSON 字符串转义（反斜杠 + 双引号）

c.e.l.infrastructure.devplan.profile.ClaudeCodeProfileGenerator#extractJsonValue(String json, String key): String
  — 轻量 JSON 字段提取（避免引入 Jackson）
```

---

## 4. 数据结构

### DevPlanProfileProperties.ClaudeCode 字段变更

```java
// v2 字段
private boolean enabled = true;
private String cliPath = "claude";
private int timeoutSeconds = 300;          // 已移除
private String skillName = "generate-project-profile";

// v3 字段
private boolean enabled = true;
private String mode = "sdk";               // 新增：sdk 或 cli
private String cliPath = "claude";
private String nodePath = "node";          // 新增
private String sdkBridgePath = "scripts/claude-code-bridge.mjs";  // 新增
private int idleTimeoutSeconds = 120;      // 新增：替代 timeoutSeconds
private int maxTimeoutSeconds = 1800;      // 新增：兜底总超时
private String skillName = "generate-project-profile";
```

### graph.yml 配置变更

```yaml
# v2
claude-code:
  enabled: true
  cli-path: claude
  timeout-seconds: 300
  skill-name: generate-project-profile

# v3
claude-code:
  enabled: true
  mode: ${DEVPLAN_CLAUDE_CODE_MODE:sdk}
  cli-path: ${DEVPLAN_CLAUDE_CODE_CLI_PATH:claude}
  node-path: ${DEVPLAN_CLAUDE_CODE_NODE_PATH:node}
  sdk-bridge-path: ${DEVPLAN_CLAUDE_CODE_SDK_BRIDGE_PATH:scripts/claude-code-bridge.mjs}
  idle-timeout-seconds: ${DEVPLAN_CLAUDE_CODE_IDLE_TIMEOUT:120}
  max-timeout-seconds: ${DEVPLAN_CLAUDE_CODE_MAX_TIMEOUT:1800}
  skill-name: generate-project-profile
```

### 桥接脚本 JSON Lines 输出协议

```json
{"type": "progress", "text": "正在扫描 src/main/java..."}
{"type": "tool_use", "tool": "Read", "input": "src/main/java/..."}
{"type": "result", "success": true, "costUsd": 0.15, "durationMs": 45000, "numTurns": 12}
{"type": "error", "message": "API rate limit exceeded"}
{"type": "final_output", "text": "完整的 assistant 文本输出"}
```

---

## 5. 重要约束与边界

- **空闲超时语义**：`lastActivityTime` 由 outputReader 线程在每次 `readLine()` 成功时更新（`AtomicLong.set`）。主线程每秒用 `System.currentTimeMillis() - lastActivityTime.get()` 计算空闲时间
- **输出消费顺序**：outputReader 线程必须在 `process.start()` 之后、`process.waitFor()` 之前启动，否则存在缓冲区死锁风险
- **进程清理**：超时后先 `destroyForcibly()`，再 `outputReader.join(3000)` 等待读取线程退出；正常结束后 `outputReader.join(5000)` 等待消费剩余输出
- **JSON 解析**：`extractJsonValue` 是轻量字符串提取，不引入 Jackson。只用于日志展示，解析失败返回空字符串，不影响功能
- **不处理的场景**：bridge 脚本内部错误的详细诊断（仅透传 exitValue 和 stdout）

---

## 6. 下游依赖调用

```text
// Node.js 子进程调用
ProcessBuilder → node scripts/claude-code-bridge.mjs '<json-config>'
  → @anthropic-ai/claude-code SDK → Anthropic API

// CLI 子进程调用（备选）
ProcessBuilder → claude --print --dangerously-skip-permissions --output-format text '<prompt>'
```

---

## 7. 异常处理要点

| 场景 | 处理 |
|------|------|
| Node.js 不存在 | `isSdkAvailable()=false`，Chain 跳过 |
| 桥接脚本不存在 | `isSdkAvailable()=false`，Chain 跳过 |
| SDK exitValue != 0 | log.error + 返回 empty |
| 空闲超时 | destroyForcibly + log.error（含空闲时长和阈值）+ empty |
| 总超时 | destroyForcibly + log.error（含总时长）+ empty |
| IOException | log.error + empty |
| InterruptedException | Thread.currentThread().interrupt() + empty |
| outputReader 线程 IOException | log.warn，不影响主流程 |
| profile.md 未生成 | log.warn + empty |
