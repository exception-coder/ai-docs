# 编码摘要文档

> 本文档由完整设计文档精简而来，供 AI 辅助编码时使用，聚焦实现所需的最小必要信息。
> 对应完整文档：`大文本块提取工具-20260402-v1.md`

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-02 | 初始版本 |

---

## 1. 核心业务规则

- 起始标识匹配：`line.contains(startMarker)` 为 true 时新建缓冲区，写入当前行；若上一个 block 尚未关闭则丢弃
- 结束标识匹配：`line.contains(endMarker)` 为 true 时将当前行追加后触发回调，重置缓冲区
- 文件末尾未关闭的 block：丢弃，记录 DEBUG 日志，不抛出异常
- 编码：固定 UTF-8
- 行分隔符：每行末尾追加 `'\n'`

---

## 2. 接口契约

无 HTTP 接口，纯工具类静态方法。

### 关键方法签名（全类名）

```java
com.exceptioncoder.llm.infrastructure.util.TextBlockExtractor#extractFromFile(Path filePath, String startMarker, String endMarker, Consumer<String> handler): long
com.exceptioncoder.llm.infrastructure.util.TextBlockExtractor#extractFromText(String text, String startMarker, String endMarker): List<String>
com.exceptioncoder.llm.infrastructure.util.TextBlockExtractor#doExtract(BufferedReader reader, String startMarker, String endMarker, Consumer<String> handler): long  // private
```

---

## 3. 涉及类清单（全类名）

| 全类名 | 操作 | 说明 |
|--------|------|------|
| `com.exceptioncoder.llm.infrastructure.util.TextBlockExtractor` | 新建 | 大文本块流式提取工具类 |

---

## 4. 数据结构

不涉及数据库。无 DTO/DO。

---

## 5. 重要约束与边界

- 无共享可变状态，线程安全
- 单个 block 体积无上限限制，调用方自行注意内存
- `extractFromFile` 声明 `throws IOException`，不内部捕获
- `extractFromText` 内部捕获 IOException（StringReader 不会实际抛出），出现时 log.error 并返回空列表

---

## 6. 下游依赖调用

无外部依赖，仅 JDK 标准库 + SLF4J。

---

## 7. 异常处理要点

- 文件读取 I/O 异常 → 向调用方抛出 `IOException`
- StringReader 意外 IOException → `log.error("描述: {}", e.getMessage(), e)`，返回空列表
- 未关闭 block → 丢弃，`log.debug` 记录
