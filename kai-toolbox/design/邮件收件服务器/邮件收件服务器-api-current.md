# 邮件收件服务器 API 文档

> 最后更新：2026-05-06
> Base Path: `/api/mail`

---

## 1. 收件箱列表

**GET** `/api/mail/inbox`

查询收件箱邮件列表，按接收时间倒序分页。

### 请求参数（Query）

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `page` | int | 否 | 页码，从 0 开始，默认 0 |
| `size` | int | 否 | 每页数量，默认 20，最大 100 |
| `toAddress` | string | 否 | 按收件人地址过滤（精确匹配） |
| `isRead` | boolean | 否 | 过滤已读/未读；不传则返回全部 |
| `keyword` | string | 否 | 关键词模糊搜索（匹配主题、发件人） |

### 响应体

```json
{
  "items": [
    {
      "id": "uuid",
      "fromAddr": "noreply@amazon.com",
      "toAddr": "shop1@yourdomain.com",
      "subject": "Your order has shipped",
      "receivedAt": "2026-05-06T10:30:00",
      "isRead": false,
      "hasAttachment": false,
      "rawSize": 4096
    }
  ],
  "total": 128,
  "page": 0,
  "size": 20,
  "unreadCount": 5
}
```

### 响应字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 邮件唯一 ID（UUID） |
| `fromAddr` | string | 发件人地址 |
| `toAddr` | string | 收件人地址 |
| `subject` | string | 邮件主题，可能为 null |
| `receivedAt` | string | 接收时间，ISO 8601 格式 |
| `isRead` | boolean | 是否已读 |
| `hasAttachment` | boolean | 是否含附件 |
| `rawSize` | int | 原始邮件大小（字节） |
| `total` | int | 命中总数（分页用） |
| `unreadCount` | int | 本次查询条件下的未读数 |

---

## 2. 邮件详情

**GET** `/api/mail/inbox/{id}`

获取单封邮件详情，同时将该邮件标记为已读。

### 路径参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 邮件 UUID |

### 响应体

```json
{
  "id": "uuid",
  "messageId": "<abc@amazon.com>",
  "fromAddr": "noreply@amazon.com",
  "toAddr": "shop1@yourdomain.com",
  "subject": "Your order has shipped",
  "bodyText": "Your order #123 has shipped...",
  "bodyHtml": "<html>...</html>",
  "attachments": [
    {
      "filename": "invoice.pdf",
      "mimeType": "application/pdf",
      "size": 81920
    }
  ],
  "receivedAt": "2026-05-06T10:30:00",
  "isRead": true,
  "rawSize": 4096
}
```

### 错误码

| HTTP 状态 | 说明 |
|-----------|------|
| 200 | 成功 |
| 404 | 邮件不存在 |

---

## 3. 标记已读

**PATCH** `/api/mail/inbox/{id}/read`

手动将指定邮件标记为已读（获取详情时会自动标记，此接口用于批量或手动触发场景）。

### 路径参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 邮件 UUID |

### 响应体

```json
{ "success": true }
```

### 错误码

| HTTP 状态 | 说明 |
|-----------|------|
| 200 | 成功 |
| 404 | 邮件不存在 |

---

## 4. 删除邮件

**DELETE** `/api/mail/inbox/{id}`

物理删除单封邮件，不可恢复。

### 路径参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `id` | string | 邮件 UUID |

### 响应体

```json
{ "success": true }
```

### 错误码

| HTTP 状态 | 说明 |
|-----------|------|
| 200 | 成功 |
| 404 | 邮件不存在 |

---

## 5. 统计信息

**GET** `/api/mail/stats`

获取收件箱整体统计（未读数、总数）。

### 响应体

```json
{
  "total": 128,
  "unreadCount": 5
}
```
