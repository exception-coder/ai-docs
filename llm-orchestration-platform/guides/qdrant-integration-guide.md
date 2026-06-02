# Qdrant 向量存储集成说明

## 当前状态

由于 Qdrant Java Client 1.17.0 的 API 结构复杂，当前实现为简化骨架版本。

## 问题分析

Qdrant Java Client 使用 gRPC 生成的类，其包结构为：
- `io.qdrant.client.grpc.Collections` - 集合相关操作
- `io.qdrant.client.grpc.Points` - 点（向量）相关操作

但是这些类的内部结构（如 `Points.Value`、`Points.Vector` 等）可能在不同版本中有所不同。

## 推荐的解决方案

### 方案 1：使用 Qdrant 官方示例代码

1. 访问 Qdrant Java Client GitHub: https://github.com/qdrant/java-client
2. 查看 examples 目录中的示例代码
3. 根据官方示例调整实现

### 方案 2：使用 Spring AI 的 Qdrant 集成

Spring AI 提供了 Qdrant 的开箱即用集成：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-qdrant-store-spring-boot-starter</artifactId>
</dependency>
```

使用方式：
```java
@Autowired
private VectorStore vectorStore;

// 存储
vectorStore.add(documents);

// 检索
List<Document> results = vectorStore.similaritySearch(
    SearchRequest.query("查询文本").withTopK(10)
);
```

### 方案 3：等待依赖正确加载后完善实现

1. 确保 Maven 依赖已正确下载
2. 在 IDE 中刷新 Maven 项目
3. 查看 `io.qdrant.client.grpc.Points` 类的实际结构
4. 根据实际可用的类和方法完善实现

## 临时解决方案

当前代码已实现为抛出 `UnsupportedOperationException` 的骨架，不会影响项目编译和其他模块的运行。

## 下一步行动

建议采用**方案 2（Spring AI 集成）**，因为：
1. Spring AI 已经封装好了 Qdrant 的复杂 API
2. 提供统一的 VectorStore 接口
3. 自动处理向量化和序列化
4. 与项目现有的 Spring AI 技术栈一致

## 实现步骤（使用 Spring AI）

1. 添加依赖到 `llm-infrastructure/pom.xml`
2. 配置 Qdrant 连接信息
3. 注入 `VectorStore` 并实现 `VectorStoreRepository` 接口
4. 使用 Spring AI 的 Document 模型存储和检索

详细实现可参考 Spring AI 官方文档：
https://docs.spring.io/spring-ai/reference/api/vectordbs/qdrant.html

