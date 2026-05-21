# 邮件收件服务器 - DNS 部署手册

> 最后更新：2026-05-09
> 模版类型：部署手册
> 状态：可用
> 关联设计：[邮件收件服务器-current.md](邮件收件服务器-current.md)

把一个新域名接入 kai-toolbox 内嵌 SMTP 收件箱的标准操作清单。本手册以 `exception-coder.com` 为示例，替换为任意主域同样适用。

---

## 1. 前置条件

| # | 项 | 说明 |
|---|---|------|
| 1 | 一台公网可达的服务器 | 已知公网 IP，记为 `<MAIL_HOST_IP>` |
| 2 | 25 端口已解封 | 腾讯云/阿里云/AWS 默认封 25，需提工单解封 |
| 3 | 安全组 / 防火墙放行 TCP 25 入站 | 来源 `0.0.0.0/0` |
| 4 | 域名管理后台访问权限 | 能添加 A / MX 记录 |

> 这台机器同时跑 kai-toolbox 即可，无需独立邮件主机。

---

## 2. DNS 配置

只需 **2 条核心记录**（第 3 条可选）：

| # | 主机记录 | 类型 | 记录值 | 优先级 | 用途 |
|---|---------|------|--------|--------|------|
| 1 | `smtp` | A | `<MAIL_HOST_IP>` | - | smtp.exception-coder.com → 邮件服务器 |
| 2 | `@` | MX | `smtp.exception-coder.com` | `10` | 主域邮件路由到 smtp 子域 |
| 3 | `@` | A | `<WEB_HOST_IP>` | - | 主域兜底（仅当主域要解析为网站时配） |

### 关键约束

- **顺序**：必须先配第 1 条 A 记录，再配第 2 条 MX。MX 指向的域名若不能解析，对方邮件服务器会判定无效路由
- **MX 记录值不能填 IP**：RFC 5321 规定 MX 只能指向域名（FQDN）
- **「主机记录」是关键**：决定哪个域能收信
  - `@` + MX `smtp.exception-coder.com` → `任意用户名@exception-coder.com` 都能收 ✅
  - `smtp` + MX `smtp.exception-coder.com` → 只有 `任意用户名@smtp.exception-coder.com` 能收 ❌

### 邮件路由示意

```
发件方发到 kai@exception-coder.com
  ↓ DNS 查 exception-coder.com 的 MX
  → smtp.exception-coder.com (优先级 10)
  ↓ DNS 查 smtp.exception-coder.com 的 A
  → <MAIL_HOST_IP>
  ↓ TCP 连接 <MAIL_HOST_IP>:25
  → kai-toolbox SMTP 服务器接收
```

---

## 3. 服务器端配置

### 3.1 应用配置（`toolbox-starter/src/main/resources/application.yml`）

```yaml
toolbox:
  mail:
    enabled: true            # 必须改为 true
    port: 25                 # 公网邮件标准端口；Linux 非 root 用 1025 + iptables 转发
    hostname: smtp.exception-coder.com   # SMTP EHLO 响应用，建议填实际 FQDN
    sender-whitelist: []     # 留空接受所有发件人；按需加 "@amazon.com" 这种后缀
```

### 3.2 关于「接收域白名单」

**当前实现没有 RCPT TO 域名过滤**（见 [MailMessageHandler.java:56-59](../../../../d:/Users/zhang/IdeaProjects/kai-toolbox/tools/tool-mail/src/main/java/com/exceptioncoder/toolbox/mail/smtp/MailMessageHandler.java#L56-L59)），**新增域名无需改代码或配置**。任何能连进 25 端口的邮件，无论收件人是 `@exception-coder.com` 还是 `@chivepockets.com`，都会按 `to_addr` 入库。

> 安全考量：因为公网暴露 25 端口意味着任何人都能往这台机器发邮件。当前依赖网络层（域名 MX 解析）做隐式过滤——只有 MX 指过来的域才会有正常流量；脚本探测的会以无效收件人形式落库。如未来需要严格过滤，需扩展 `MailProperties` 增加 `recipient-domain-whitelist` 并在 `MailMessageHandler.recipient()` 里校验。

### 3.3 Linux 25 端口三种解决方案

| 方案 | 操作 | 适用 |
|------|------|------|
| 直接绑 25 | `sudo java -jar kai-toolbox.jar` 或 `setcap 'cap_net_bind_service=+ep'` | 简单，不推荐生产 |
| 高位端口 + iptables 转发 | 应用绑 1025；`iptables -t nat -A PREROUTING -p tcp --dport 25 -j REDIRECT --to-port 1025` | **推荐**，应用普通用户运行 |
| 反向代理 | nginx stream / haproxy 监听 25 转发 1025 | 多实例时用 |

### 3.4 监听核查

```bash
# Linux
ss -tlnp | grep ':25 '
# Windows
netstat -ano | findstr :25
```

确认监听地址是 `0.0.0.0:25` 或 `*:25`，不是 `127.0.0.1:25`，否则外部连不进来。

---

## 4. 验证清单

按顺序排查，任一环节卡住就停在那一步。

### 4.1 DNS 生效（5-10 分钟）

```powershell
nslookup -type=mx exception-coder.com
# 期望: mail exchanger = smtp.exception-coder.com, preference = 10

nslookup smtp.exception-coder.com
# 期望: Address: <MAIL_HOST_IP>
```

### 4.2 端口连通性

```powershell
Test-NetConnection -ComputerName smtp.exception-coder.com -Port 25
# 期望: TcpTestSucceeded : True
```

`telnet smtp.exception-coder.com 25` 看到 `220 ... ESMTP` 横幅即说明 SMTP 服务握手 OK。

### 4.3 真实收信

任意外部邮箱（QQ / Gmail / 163）发邮件到 `test@exception-coder.com`，再：
1. 看 kai-toolbox 后端日志，找 `MailMessageHandler` 的 `邮件入库成功` 行
2. 打开 WebUI `http://localhost:18080/tools/mail` 看收件箱列表是否出现

### 4.4 常见排查

| 症状 | 可能原因 |
|------|---------|
| 对方退回 `MX record not found` | DNS 未生效 / MX 主机记录配错（不是 `@`） |
| 对方退回 `Connection timed out` | 25 端口未解封 / 安全组未放行 / 应用未启动 |
| 对方退回 `550 relay denied` | 不是本服务返回的（本服务不做 relay 校验），是中间网关代收的，需查 ISP |
| 进对方垃圾箱（仅当反向发信时） | 缺 PTR / SPF / DKIM / DMARC，见第 5 节 |
| 日志收到连接但无入库 | RCPT TO 列表为空（见 [MailMessageHandler.java:63-66](../../../../d:/Users/zhang/IdeaProjects/kai-toolbox/tools/tool-mail/src/main/java/com/exceptioncoder/toolbox/mail/smtp/MailMessageHandler.java#L63-L66)）；或 MIME 解析异常，看 stack |

---

## 5. 附加配置：从这台机器对外发邮件

> 当前 kai-toolbox 仅作收件用，**不发邮件**。下列配置仅在未来新增发件功能时需要。

| 配置 | DNS 类型 | 主机记录 | 记录值 |
|------|---------|---------|--------|
| PTR | 反向解析 | （工单提交） | `<MAIL_HOST_IP>` ↔ `smtp.exception-coder.com` |
| SPF | TXT | `@` | `v=spf1 a mx ~all` |
| DKIM | TXT | `default._domainkey`（密钥名按邮件服务定） | `v=DKIM1; k=rsa; p=<公钥>` |
| DMARC | TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:postmaster@exception-coder.com` |

不配这四样，发出去的邮件大概率被对方判垃圾或直接拒收。

---

## 6. 多域名共存

`kai-toolbox` 单实例支持任意数量的接收域，操作上：

1. 给每个新域配第 2 节的 1+2 两条 DNS 记录，**A 记录值都指同一台机器 IP**
2. 应用配置无需任何改动
3. 邮件按 `to_addr` 入库，前端按 to_addr 过滤分组（[MailController.java](../../../../d:/Users/zhang/IdeaProjects/kai-toolbox/tools/tool-mail/src/main/java/com/exceptioncoder/toolbox/mail/api/MailController.java) 已支持 `toAddress` 查询参数）

例如 `chivepockets.com` 和 `exception-coder.com` 同时收信：

```
chivepockets.com    MX → smtp.chivepockets.com    → 170.106.186.65:25
exception-coder.com MX → smtp.exception-coder.com → 170.106.186.65:25
                                                     ↓
                                              同一个 kai-toolbox 实例
                                                     ↓
                                          mail_inbox 表按 to_addr 区分
```

---

## 7. 操作顺序总览

1. **DNS**：smtp 子域 A 记录 → @ 主域 MX 记录 → 等 5 分钟生效
2. **DNS 验证**：`nslookup -type=mx` 通过
3. **服务器**：解封 25 端口 + 安全组放行 + 应用 `enabled: true`
4. **端口验证**：`Test-NetConnection -Port 25` 通过
5. **真实测试**：外部邮箱发信 → 看日志 + WebUI

---

## 附录：示例域名实际值

| 字段 | chivepockets.com（已配） | exception-coder.com（本次） |
|------|--------------------------|----------------------------|
| `<MAIL_HOST_IP>` | 170.106.186.65 | 待填 |
| smtp 子域 A | smtp.chivepockets.com → 170.106.186.65 | smtp.exception-coder.com → 待填 |
| @ 主域 MX | smtp.chivepockets.com / 10 | smtp.exception-coder.com / 10 |
| 主域 A（可选） | 8.138.90.250 | 待填 / 不配 |
