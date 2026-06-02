# 编码违规记录

> 本文件由 `coding-violation-log` Skill 自动维护。
> AI 编码前必须读取本文件，避免重犯已记录的错误。

| # | 类型 | 违规描述 | 正确做法 | 涉及文件 | 首次发生 | 次数 |
|---|------|---------|---------|---------|---------|------|
| 1 | 分层依赖 | Application 层（ScanNode）直接 import 了 Infrastructure 层的 ProfileGeneratorChain、ProfileMarkdownReader、ProfileIndexTool、DevPlanProfileProperties | 在 Domain 层定义接口（ProfileGeneratorService、ProfileReader、ProfileIndexService、ProfileCacheStrategy），Infrastructure 层实现，Application 层只依赖 Domain 接口 | ScanNode.java | 2026-04-12 | 1 |
