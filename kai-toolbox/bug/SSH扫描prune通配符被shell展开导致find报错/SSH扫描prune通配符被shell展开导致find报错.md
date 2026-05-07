# SSH 扫描 prune 通配符被 shell 展开导致 find 报错

## 问题背景

- 功能路径：treesize → SSH 主机扫描（`startSshScan`）
- 现象：对远程 Linux 系统执行 SSH 扫描时（尤其是扫描根目录 `/`），扫描任务抛出 `IOException`，错误来自远程 `find` 命令的 stderr
- 复现参数：

```json
{
  "sourceType": "SSH",
  "sshHostId": "<任意已保存主机>",
  "path": "/"
}
```

- 错误日志：

```
04:43:39.899 ERROR [treesize-scan-3] c.e.t.treesize.service.ScanService
  - ssh scan dcb940f5-690d-4d17-aa49-be5736011da8 failed for root@170.106.186.65:22
java.io.IOException: find: paths must precede expression: `/proc/11'
find: possible unquoted pattern after predicate `-path'?
    at com.exceptioncoder.toolbox.treesize.service.RemoteScanEngine.scan(RemoteScanEngine.java:61)
    at com.exceptioncoder.toolbox.treesize.service.ScanService.runSshScan(ScanService.java:172)
    ...
```

## 触发条件

| 条件 | 取值 |
|---|---|
| 扫描类型 | SSH 远程扫描 |
| 扫描路径 | 扫描路径祖先包含 `/proc`、`/sys`、`/dev`、`/run` 的路径（包括 `/`） |
| 远程系统 | 任意 Linux（`/proc` 目录下有子目录即可触发） |
| 根路径为 `/` 时 | 必现 |
| 根路径为 `/var/log` 等时 | 不触发（`/proc` 不在子树内，shell 展开为空，GNU find 忽略空 glob） |

## 涉及类清单

| 角色 | 全类名 |
|---|---|
| 远程扫描引擎（Bug 所在） | `com.exceptioncoder.toolbox.treesize.service.RemoteScanEngine` |
| SSH 连接工厂 | `com.exceptioncoder.toolbox.treesize.service.SshClientFactory` |
| 扫描任务调度 | `com.exceptioncoder.toolbox.treesize.service.ScanService` |

## 关键代码路径

| 描述 | 文件路径 | 行号 | 说明 |
|---|---|---|---|
| **Bug 根源**：prune 表达式中通配符未加引号 | `tools/tool-treesize/.../service/RemoteScanEngine.java` | 92-93 | **`/proc/*` 等 glob 在 shell 中展开为实际路径列表** |
| find 命令拼接 | `tools/tool-treesize/.../service/RemoteScanEngine.java` | 90-96 | `scanCommand()` 方法，rootPath 已正确引号，但 prune 段未处理通配符 |
| stderr 抛出点 | `tools/tool-treesize/.../service/RemoteScanEngine.java` | 59-62 | `exitStatus != 0` 时把 stderr 包成 `IOException` 抛出 |
| shell 引号工具 | `tools/tool-treesize/.../service/RemoteScanEngine.java` | 98-100 | `shellQuote()` 用单引号包裹，存在但未应用于 prune 段 |

## 核心流程分析

### 时序图（错误路径）

```mermaid
sequenceDiagram
    participant Service as "ScanService"
    participant Engine as "RemoteScanEngine"
    participant JSch as "JSch ChannelExec"
    participant Shell as "远程 /bin/sh"
    participant Find as "GNU find"

    Service->>Engine: scan(host, "/", ...)
    Engine->>Engine: scanCommand("/")
    Note over Engine: 生成命令中含未引号的 /proc/* 等
    Engine->>JSch: setCommand(cmd)
    JSch->>Shell: exec 命令字符串
    Shell->>Shell: glob 展开 /proc/* → /proc/1 /proc/2 ... /proc/11
    Shell->>Find: find -P '/' \\( -path /proc -o -path /proc/1 -o /proc/2 ... \\) -prune -o ...

    rect rgb(255, 220, 220)
        Note over Find: 多余的 /proc/N 被当作\n额外的 PATH 参数
        Find-->>Shell: 退出码 1 + stderr
    end

    Shell-->>JSch: exit 1
    JSch-->>Engine: exitStatus=1, err="find: paths must precede expression..."
    Engine-->>Service: throw IOException
```

### 流程图（prune 通配符展开决策树）

```mermaid
flowchart TD
    A["scanCommand(rootPath)"] --> B["shellQuote(rootPath) → 正确引号"]
    A --> C["拼接 prune 字符串\n含 /proc/* /sys/* /dev/* /run/*"]
    C --> D{"shell 执行时\n/proc 是否存在且非空?"}
    D -->|"是（Linux 系统）"| E["shell glob 展开\n/proc/* → /proc/1 /proc/2 ... /proc/N"]
    D -->|"否"| F["glob 展开为空字符串\n或被忽略（无影响）"]
    E --> G["find 接收到多余位置参数\n/proc/1 /proc/2 ... 被当作 PATH"]
    G --> H["find 报错：paths must precede expression"]
    H --> I["exitStatus = 1\nstderr 包成 IOException 抛出"]
    F --> J["find 正常执行"]

    style E fill:#ffe0e0
    style G fill:#ffe0e0
    style H fill:#ffcccc
    style I fill:#ffcccc
```

### 泳道图（分层职责）

```mermaid
flowchart LR
    subgraph "Java 层"
        A1["RemoteScanEngine\nscanCommand() 生成命令字符串"]
        A2["JSch ChannelExec\n透传命令到远程 shell"]
    end
    subgraph "远程 Shell 层"
        B1["sh 解析命令字符串"]
        B2["glob 展开 /proc/*\n→ 多个路径 token"]
    end
    subgraph "GNU find 层"
        C1["接收展开后的参数列表"]
        C2["检测到位置参数在表达式之后"]
        C3["stderr 输出错误并以 1 退出"]
    end
    A1 -->|"含未引号 glob"| A2
    A2 -->|"原始字符串"| B1
    B1 --> B2
    B2 -->|"多余 PATH 参数"| C1
    C1 --> C2 --> C3
    C3 -->|"exitStatus=1 + stderr"| A2
    A2 -->|"throw IOException"| A1
```

## 相关代码 / SQL 清单

**当前错误的 prune 字符串**（`RemoteScanEngine.java:92-95`）：

```java
private static String scanCommand(String rootPath) {
    String quoted = shellQuote(rootPath);
    String prune = "\\( -path /proc -o -path /proc/* -o -path /sys -o -path /sys/* "
            + "-o -path /dev -o -path /dev/* -o -path /run -o -path /run/* \\) -prune -o";
    return "LC_ALL=C find -P " + quoted + " " + prune
            + " -printf '%y\\037%s\\037%b\\037%T@\\037%p\\0'";
}
```

**实际发往远程的命令（根路径 `/` 时）**：

```bash
LC_ALL=C find -P '/' \( -path /proc -o -path /proc/* -o -path /sys ...
```

**shell 展开后 find 实际接收的参数（示意）**：

```bash
find -P '/' \( -path /proc -o -path /proc/1 /proc/2 /proc/3 ... /proc/11 -o -path /sys ...
```

`/proc/1`、`/proc/2`、...、`/proc/11` 被 `find` 识别为额外的 PATH 参数，而此时表达式 `\(` 已经开始，触发错误。

## 根因总结

| 问题现象 | 根因 |
|---|---|
| `find: paths must precede expression` | `prune` 表达式中 `/proc/*` 等 glob 未加 shell 引号，`/bin/sh` 在执行前将其展开为 `/proc` 下实际存在的目录列表，多余的路径 token 被 `find` 当作 PATH 参数插入表达式内部 |
| 仅扫描 `/` 时必现 | 扫描子路径时 `/proc` 不在子树内，`/proc/*` 展开为空（bash noglob 时保留字面量），GNU find 不报错；扫描 `/` 时 `/proc` 必然存在且有内容 |

## 修复方案

### 短期（治标）

对 `prune` 字符串中所有 glob 模式加单引号，防止 shell 展开：

```java
String prune = "\\( -path /proc -o -path '/proc/*' -o -path /sys -o -path '/sys/*' "
        + "-o -path /dev -o -path '/dev/*' -o -path /run -o -path '/run/*' \\) -prune -o";
```

`find` 的 `-path` 谓词自身支持 glob 语法（`*`、`?`、`[...]`），加单引号后 shell 不展开，`find` 内部正确用通配符匹配路径。

### 中期（治本）

`-path /proc` 已经覆盖了 `/proc` 目录本身，`-prune` 会阻止 `find` 继续进入该目录，所以 `-path /proc/*` 实际上是冗余的（`find` 不会进入 `/proc` 去产生 `/proc/*` 的输出）。可以简化为：

```java
String prune = "\\( -path /proc -o -path /sys -o -path /dev -o -path /run \\) -prune -o";
```

这样彻底消除 glob，命令更简洁，无 shell 展开风险。

### 配置 / 运维

临时规避：扫描时不使用 `/` 作为根路径，改用 `/home`、`/data` 等不含 `/proc` 子树的路径，可绕过此 bug。
