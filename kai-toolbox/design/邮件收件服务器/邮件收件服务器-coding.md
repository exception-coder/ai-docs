# 邮件收件服务器 — 编码摘要

> 最后更新：2026-05-06
> 对应设计文档：邮件收件服务器-current.md
> 接口契约：邮件收件服务器-api-current.md

---

## 1. 核心业务规则

- 发件人白名单：`toolbox.mail.sender-whitelist` 为空时接受所有；非空时 `sender.contains(item)` 匹配（支持 `@amazon.com` 域名前缀）
- RCPT TO 多收件人：每个收件人独立一行入库，body_text/body_html 复制存储
- body 存储：`text/plain` → body_text；`text/html` → body_html；`multipart/alternative` 两者都存
- 附件只存元数据：JSON 串存 attachments 列，不落盘文件内容
- 打开详情自动标记已读（GET /inbox/{id} 内部调用 markRead）
- 删除为物理删除
- 邮件 id：UUID，非 SMTP Message-ID

---

## 2. 接口入口指针

> 字段级契约见 邮件收件服务器-api-current.md

| 方法 | 路径 | 实现 |
|------|------|------|
| GET | `/api/mail/inbox` | `MailController#listInbox` |
| GET | `/api/mail/inbox/{id}` | `MailController#getDetail` |
| PATCH | `/api/mail/inbox/{id}/read` | `MailController#markRead` |
| DELETE | `/api/mail/inbox/{id}` | `MailController#deleteById` |
| GET | `/api/mail/stats` | `MailController#getStats` |

---

## 3. 涉及类清单

### 3.1 `com.exceptioncoder.toolbox.mail.config.MailProperties`

ConfigurationProperties，前缀 `toolbox.mail`。

```java
@ConfigurationProperties("toolbox.mail")
public class MailProperties {
    private boolean enabled = true;
    private int port = 25;
    private String hostname = "localhost";
    private List<String> senderWhitelist = List.of(); // 空=接受所有
}
```

### 3.2 `com.exceptioncoder.toolbox.mail.config.SmtpServerManager`

```java
@PostConstruct
public void start()          // 构建 SMTPServer，注册 MessageHandlerFactory，按 enabled 决定是否 start
@PreDestroy
public void stop()           // smtpServer.stop()
```

- MessageHandlerFactory：每次 SMTP 连接 `new MailMessageHandler(mailInboxRepository, mailProperties)`
- SMTPServer 设置：`setHostName`、`setPort`、不开 TLS

### 3.3 `com.exceptioncoder.toolbox.mail.smtp.MailMessageHandler`

implements `org.subethamail.smtp.MessageHandler`

```java
// 字段
private final MailInboxRepository repo;
private final MailProperties props;
private String fromAddr;
private final List<String> toAddrs = new ArrayList<>();

void from(String from)                // 白名单校验，失败抛 RejectException
void recipient(String to)             // toAddrs.add(to)
void data(InputStream stream)         // 解析 MimeMessage，每个 toAddr 入库一行
void done()                           // log

// 私有方法
private MailInbox buildInbox(MimeMessage msg, String toAddr)
private String extractTextBody(Object content, String contentType)
private String extractHtmlBody(Object content, String contentType)
private List<MailAttachment> extractAttachments(MimeMultipart mp)
private boolean isSenderAllowed(String sender)
```

### 3.4 `com.exceptioncoder.toolbox.mail.domain.MailInbox`

```java
@Data @Builder
public class MailInbox {
    String id;           // UUID
    String messageId;    // SMTP Message-ID header
    String fromAddr;
    String toAddr;
    String subject;
    String bodyText;
    String bodyHtml;
    String attachments;  // JSON 串，List<MailAttachment> 序列化结果
    LocalDateTime receivedAt;
    boolean read;
    Long rawSize;
}
```

### 3.5 `com.exceptioncoder.toolbox.mail.domain.MailAttachment`

```java
@Data @Builder
public class MailAttachment {
    String filename;
    String mimeType;
    long size;
}
```

### 3.6 `com.exceptioncoder.toolbox.mail.repository.MailInboxRepository`

JdbcTemplate 操作，对应 `mail_inbox` 表。

```java
void save(MailInbox mail)
Optional<MailInbox> findById(String id)
List<MailInbox> findPage(MailInboxFilter filter, int page, int size)
int countTotal(MailInboxFilter filter)
int countUnread(MailInboxFilter filter)
void markRead(String id)
void deleteById(String id)
```

Filter 字段：`toAddress`（精确）、`isRead`（nullable Boolean）、`keyword`（LIKE 模糊，匹配 subject + from_addr）

### 3.7 `com.exceptioncoder.toolbox.mail.controller.MailController`

```java
@GetMapping("/api/mail/inbox")
ResponseEntity<MailListResponse> listInbox(
    @RequestParam(defaultValue="0") int page,
    @RequestParam(defaultValue="20") int size,
    @RequestParam(required=false) String toAddress,
    @RequestParam(required=false) Boolean isRead,
    @RequestParam(required=false) String keyword)

@GetMapping("/api/mail/inbox/{id}")
ResponseEntity<MailDetailDTO> getDetail(@PathVariable String id)   // 内部调 markRead

@PatchMapping("/api/mail/inbox/{id}/read")
ResponseEntity<Map<String,Boolean>> markRead(@PathVariable String id)

@DeleteMapping("/api/mail/inbox/{id}")
ResponseEntity<Map<String,Boolean>> deleteById(@PathVariable String id)

@GetMapping("/api/mail/stats")
ResponseEntity<MailStatsDTO> getStats()
```

### 3.8 `com.exceptioncoder.toolbox.mail.config.MailToolDescriptor`

```java
// id()      → "mail"
// name()    → "收件箱"
// icon()    → "mail"
// route()   → "/tools/mail"
// group()   → "网络工具"
// order()   → 30
```

---

## 4. 数据结构

**mail-schema.sql**（放 `src/main/resources/db/mail-schema.sql`，SchemaInitializer 自动加载）：

```sql
CREATE TABLE IF NOT EXISTS mail_inbox (
    id          TEXT    PRIMARY KEY,
    message_id  TEXT,
    from_addr   TEXT    NOT NULL,
    to_addr     TEXT    NOT NULL,
    subject     TEXT,
    body_text   TEXT,
    body_html   TEXT,
    attachments TEXT,
    received_at TEXT    NOT NULL,
    is_read     INTEGER NOT NULL DEFAULT 0,
    raw_size    INTEGER
);
CREATE INDEX IF NOT EXISTS idx_mail_to_addr     ON mail_inbox(to_addr);
CREATE INDEX IF NOT EXISTS idx_mail_received_at ON mail_inbox(received_at DESC);
CREATE INDEX IF NOT EXISTS idx_mail_is_read     ON mail_inbox(is_read);
```

**tool-mail/pom.xml 新增依赖：**

```xml
<dependency>
    <groupId>org.subethamail</groupId>
    <artifactId>subethasmtp</artifactId>
    <version>3.1.7</version>
</dependency>
<dependency>
    <groupId>com.sun.mail</groupId>
    <artifactId>jakarta.mail</artifactId>
    <version>2.0.1</version>
</dependency>
```

---

## 5. 重要约束与边界

| 约束 | 说明 |
|------|------|
| 端口权限 | Linux 25 端口需 root 或 `setcap`；建议生产用 1025，DNS/iptables 转发 |
| body 截断 | body_text / body_html 超过 2MB 时截断，在末尾追加 `\n[内容已截断]` |
| jakarta.mail 包名 | SubEtha 3.1.7 内部使用 `javax.mail`；MimeMessage 解析用 `com.sun.mail:jakarta.mail`（提供 `jakarta.mail.*`），两者共存不冲突 |
| 前端 XSS | body_html 在 iframe sandbox 中渲染，属于前端约束，后端不做 HTML 清洗 |
| DTO 转换 | MailInbox → MailListItemDTO / MailDetailDTO 在 Controller 内 static 方法转换，不引入额外 mapper |
| 事务 | 单行 INSERT，无需事务；多收件人循环 save，异常时已入库的行不回滚（邮件送达语义） |
