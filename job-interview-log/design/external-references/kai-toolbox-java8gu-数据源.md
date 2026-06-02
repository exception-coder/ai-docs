# 跨项目引用 · kai-toolbox java8gu 数据源

> 本文件只是外部引用占位 —— kai-toolbox 的 `java8gu` 模块改造为从 job-interview-log 仓库的 `java8gu-速记版/` 目录在线拉取目录与文件。
>
> 设计文档主体在 kai-toolbox 知识库：
>
> **`C:\Users\张凯\Documents\ai-docs\kai-toolbox\design\java8gu-github数据源\java8gu-github数据源-current.md`**

## 简介

| 项 | 值 |
|---|---|
| 消费方 | kai-toolbox / frontend / `src/features/java8gu` |
| 数据源仓库 | `https://github.com/exception-coder/JobInterviewLog` |
| 数据源目录 | `java8gu-速记版/` |
| 接入方式 | 浏览器端 Trees API + raw.githubusercontent.com 直拉 |
| 对本仓的影响 | **无源码改动**，仅作为只读上游 |
