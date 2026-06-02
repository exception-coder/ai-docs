# 项目快速运行器 plugin · 编码摘要

> 最后更新：2026-05-31
> 配套设计：[项目快速运行器plugin-current.md](./项目快速运行器plugin-current.md)

变更摘要：新建 plugin 源码 repo `D:\Users\zhang\IdeaProjects\project-runner-plugin\`。包含 1 个 SKILL.md + 6 个 slash command + 1 个 Codex AGENTS.md 片段 + 1 个示例 yml + plugin.json + README。

---

## 1. 核心业务规则（编码必读）

- **配置路径硬编码**：`~/.claude/project-runner/projects.yml`，所有命令都 Read 这一个文件。Windows 上 `~` 展开为 `$HOME` 或 `$USERPROFILE`，AI 在调 Bash 时优先用 `$HOME`，命令模板里也始终写 `$HOME/.claude/project-runner/...`
- **PID 文件原子写**：写入流程是 `Write .pid.tmp → mv` 或 PowerShell `Move-Item -Force`。**禁止**直接 Write 覆盖
- **跨平台 kill**：检测平台后分支；Bash tool 自动选 PowerShell/bash，但 kill 命令本身要适配
- **fuzzy 匹配**：忽略大小写 + 全部空格 + substring；如 "kai-toolbox后端" / "kai-toolbox 后端" / "kai toolbox backend" 都应能命中 yml 里的 `kai-toolbox 后端`
- **slash command 不接收完整名称时容错**：允许 `/run-start kai-toolbox 后端`、`/run-start "kai-toolbox 后端"`、`/run-start kai`（fuzzy）三种写法
- **start 必须 tee 落盘**：完整命令模板 `cd "{path}" && {start} 2>&1 | tee "{HOME}/.claude/project-runner/logs/{name}.log"`，**禁止**只 `tee` 不 `2>&1`（漏 stderr）
- **ready_signal 匹配**：每收到一行 stdout 就 regex test；优先用户 yml 里的 ready_signal，否则按 §5.2 内置词表逐个尝试

---

## 2. 接口入口指针

本 plugin 无 HTTP 接口。Slash command 的"入口" = `commands/<cmd>.md`，由 Claude Code 直接读取 + 渲染：

| 命令 | 文件 |
|---|---|
| `/run-list` | `commands/run-list.md` |
| `/run-start <name>` | `commands/run-start.md` |
| `/run-stop <name>` | `commands/run-stop.md` |
| `/run-restart <name>` | `commands/run-restart.md` |
| `/run-build <name>` | `commands/run-build.md` |
| `/run-logs <name>` | `commands/run-logs.md` |

自然语言入口 = `skills/project-runner/SKILL.md`，由 Claude Code 根据 description 决定是否激活。

---

## 3. 文件清单

### `D:\Users\zhang\IdeaProjects\project-runner-plugin\`

#### `.claude-plugin/plugin.json`
JSON：

```json
{
  "name": "project-runner",
  "version": "0.1.0",
  "description": "通过 ~/.claude/project-runner/projects.yml 配置项目清单，提供 /run-start /run-stop /run-restart /run-build /run-logs /run-list 等命令",
  "author": "kpay-group/ping.yang",
  "homepage": "https://example.com/project-runner-plugin"
}
```

#### `skills/project-runner/SKILL.md`
Frontmatter + body：

- frontmatter: `name: project-runner` / `description: 用户用自然语言或 slash command 触发"启动/停止/重启/编译/查日志"且涉及 ~/.claude/project-runner/projects.yml 里登记的项目时激活...`
- body: 触发清单 + 配置读取流程 + fuzzy 匹配规则 + 命令分发 + ready_signal 内置词表

#### `commands/run-list.md`
- 描述 + body：步骤化指令告诉 AI 怎么读 yml + state/，组装一个"项目 / 状态 / PID / 启动时间"的表
- 状态判定：state/<name>.pid 存在且 PID 仍在 → RUNNING；其它 → STOPPED

#### `commands/run-start.md`
关键步骤：
1. 参数: `$1` = project name hint
2. Read `~/.claude/project-runner/projects.yml`
3. fuzzy match → ProjectConfig{path, start, ready_signal?}
4. Read `state/<name>.pid` 检查是否已运行（用 Bash `ps -p <pid>` / `tasklist /FI "PID eq <pid>"` 验证 PID 还活着）
5. 已运行 → 提示用户改用 restart 或先 stop
6. 否则：
   - 确保 `~/.claude/project-runner/logs/` 存在
   - 执行 `Bash(run_in_background=true, command="cd '${path}' && ${start} 2>&1 | tee '$HOME/.claude/project-runner/logs/${name}.log'")` —— 拿到 shell_id
   - 等 1 秒让 shell 真正起来，用 `BashOutput(shell_id)` 或日志文件首行确认进程已写出 PID（实际 Bash tool 返回的 shell_id 即 process group 句柄）
   - **写 PID 文件**：内容是 JSON `{"shell_id":"...","started_at":"...","command":"...","name":"..."}`；原子写
   - `Monitor(shell_id)` 订阅 stdout，按 ready_signal regex 监听；命中 → 停止 monitor → 告知用户
   - 兜底：30 秒无 stderr 或日志文件最近 5 秒无新增 → 视为已启动

#### `commands/run-stop.md`
关键步骤：
1. 参数 `$1` = name hint
2. Read yml + state/<name>.pid → shell_id
3. 优先 `KillShell(shell_id)`；失败用 Bash 跨平台 kill
4. 删除 state/<name>.pid
5. 告知用户

#### `commands/run-restart.md`
- 直接复用 stop + start 两个流程：先调用 stop 的指令块，再调用 start 的指令块
- 若 yml 里有 `restart` 字段，**改为**：直接跑 restart 命令（不走 stop+start）—— ready_signal 同 start

#### `commands/run-build.md`
1. 参数 `$1` = name
2. Read yml → ProjectConfig.build
3. build 为空 → 告诉用户"未配置 build 命令"
4. `Bash(command="cd '${path}' && ${build}", timeout=600_000)` 同步阻塞
5. 把 tail 50 行 stdout 返回；非 0 退出码用 ❌ 标记（注：emoji 在 plugin 文本里允许，但不写进生成代码）

#### `commands/run-logs.md`
1. 参数 `$1` = name；可选 `$2` = 行数（默认 100）；`$3` = follow 秒数（默认 0）
2. Read logs/<name>.log
3. 用 `Bash("tail -n ${lines} '$HOME/.claude/project-runner/logs/${name}.log'")` 或 PowerShell `Get-Content -Tail`
4. follow 秒数 > 0 → 用 `Bash(run_in_background=true, "tail -F -n 0 ...")` + `Monitor` + 计时后 `KillShell`
5. 把内容贴给用户

#### `codex/AGENTS.md`
Codex 用纯文本指令（无 frontmatter）：

```
# project-runner（Codex 集成）

当用户消息出现"启动/停止/重启/编译/看日志"动作 + 项目名时，按以下步骤：

1. Read ~/.claude/project-runner/projects.yml
2. fuzzy match 项目名（忽略大小写 + 去空格 + substring）
3. 多个命中让用户选；零命中告诉用户已配置项目列表
4. 根据动作执行对应流程（见下）

## 启动
... (省略，本文件后续补充)
```

#### `examples/projects.yml.example`
带详细中文注释的样例 yml。

#### `README.md`
人类可读：安装 / 配置示例 / 命令速查 / 限制 / FAQ。

---

## 4. 重要约束与边界

- **不要在 plugin 里写 Python / Node / 任何脚本** —— plugin 是 AI 指令集合，全部能力由 AI 的工具调用提供。SKILL.md 和 commands/*.md 都是描述"AI 该怎么做"的 markdown
- **`~` 在 Bash 命令字符串里展开**：实际写命令时统一用 `$HOME`，因为 Bash 工具的 shell 在不同平台 `~` 行为不一致（PowerShell 不会自动展开）
- **PID 文件 JSON 格式固定**：
  ```json
  {
    "name": "kai-toolbox 后端",
    "shell_id": "bash_1",
    "started_at": "2026-05-31T15:30:00+08:00",
    "command": "mvn -pl toolbox-starter -am spring-boot:run",
    "log_path": "/c/Users/张凯/.claude/project-runner/logs/kai-toolbox 后端.log"
  }
  ```
- **fuzzy 算法**：normalize = lowercase + trim + remove whitespace；hit if `normalize(yml_name).includes(normalize(hint))` 或 `normalize(hint).includes(normalize(yml_name))`
- **README 必须明确**：进程随 Claude Code 会话退出；要后台守护用 nohup / pm2，plugin 不接管
- **测试要点**（README 末尾 Manual checklist）：
  - 新机器首次用 `/run-list`：检测到 yml 不存在 + 提示拷贝 example
  - 启动 maven 项目 → ready_signal 命中 → 告知用户
  - 启动 npm 项目 → ready_signal 命中 → 告知用户
  - 启动一个故意不存在 path 的项目 → 报错
  - 同名重复 start → 提示已在运行
  - stop 杀掉长驻进程 → ps 看不到 + PID 文件删
  - restart → stop + start，新 shell_id
  - build npm 项目 → tail 输出 + 退出码 0
  - logs follow → 持续看到新行 30 秒后退出
  - 模糊匹配："kai" 匹配两条（前端/后端）→ 让用户选
- **跨平台**：
  - Windows: Bash tool 用 PowerShell；kill 用 `taskkill /PID {pid} /T /F`；路径用反斜杠原样保留，单引号包裹
  - macOS/Linux: bash；kill `kill -TERM` 等 3 秒 `kill -9`
  - 在 SKILL.md / 各 command.md 里用 conditional 描述："如果是 Windows / 否则"，让 AI 自己选

---

## 5. 数据结构

唯一数据结构：

### 5.1 配置 yml schema

见设计文档 §5.1。

### 5.2 PID 文件 JSON schema

```json
{
  "name": "string",
  "shell_id": "string",
  "started_at": "ISO-8601",
  "command": "string",
  "log_path": "string"
}
```
