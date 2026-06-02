# Prompt 模板管理指南

## 概述

Prompt 模板已从代码中迁移到 MySQL 数据库，支持动态管理和更新，无需重启服务。

## 数据库表结构

```sql
CREATE TABLE prompt_template (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    template_name VARCHAR(100) NOT NULL UNIQUE,
    category VARCHAR(50),
    template_content TEXT NOT NULL,
    description VARCHAR(500),
    variables TEXT,
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    version INT NOT NULL DEFAULT 1,
    created_at DATETIME NOT NULL,
    updated_at DATETIME NOT NULL,
    created_by VARCHAR(50),
    updated_by VARCHAR(50)
);
```

## 初始化数据库

1. 创建数据库和表：

```bash
mysql -u root -p < docs/init-database.sql
```

2. 配置数据库连接（`application.yml`）：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/llm_platform
    username: root
    password: your-password
```

## 模板变量语法

模板中使用 `{variableName}` 作为占位符，运行时会被实际值替换。

示例：
```
用户问题：{userQuestion}
平台特点：{platformCharacteristics}
```

## API 使用

### 1. 获取所有模板

```bash
curl http://localhost:8080/api/v1/prompt-templates
```

### 2. 获取指定模板

```bash
curl http://localhost:8080/api/v1/prompt-templates/content-optimization
```

### 3. 根据分类获取模板

```bash
curl http://localhost:8080/api/v1/prompt-templates/category/content
```

### 4. 创建或更新模板

```bash
curl -X POST http://localhost:8080/api/v1/prompt-templates \
  -H "Content-Type: application/json" \
  -d '{
    "templateName": "my-template",
    "category": "custom",
    "templateContent": "你是一个{role}。请回答：{question}",
    "description": "自定义模板示例"
  }'
```

### 5. 删除模板

```bash
curl -X DELETE http://localhost:8080/api/v1/prompt-templates/my-template
```

## 缓存机制

为了提升性能，模板查询结果会被缓存：

- **缓存时间**：30 分钟
- **缓存容量**：1000 个模板
- **缓存策略**：LRU（最近最少使用）

当模板被创建、更新或删除时，缓存会自动清空。

## 模板分类

建议的模板分类：

| 分类 | 说明 | 示例 |
|------|------|------|
| content | 内容创作 | 内容优化、文案生成 |
| chat | 对话交互 | 简单对话、客服问答 |
| development | 开发辅助 | 代码审查、代码生成 |
| analysis | 数据分析 | 文本分析、情感分析 |
| translation | 翻译转换 | 语言翻译、格式转换 |

## 最佳实践

### 1. 模板命名规范

- 使用小写字母和连字符
- 名称要有描述性
- 示例：`content-optimization`、`code-review`、`sentiment-analysis`

### 2. 变量命名规范

- 使用驼峰命名法
- 名称要清晰明确
- 示例：`{userQuestion}`、`{platformCharacteristics}`、`{originalContent}`

### 3. 模板内容编写

```
# 好的模板示例
你是一位{role}专家。

任务：{task}

要求：
1. {requirement1}
2. {requirement2}

输出格式：{outputFormat}
```

### 4. 版本管理

- 重大修改时增加版本号
- 保留历史版本用于回滚
- 在 description 中记录变更说明

### 5. 测试模板

创建新模板后，建议：
1. 使用不同的变量值测试
2. 验证输出格式是否正确
3. 检查边界情况

## 模板示例

### 内容优化模板

```
你是一位专业的内容创作优化专家。

平台特点：{platformCharacteristics}
内容风格：{styleDescription}
原始内容：{originalContent}

请优化内容，输出JSON格式：
{
  "optimizedContent": "优化后内容",
  "suggestedTitles": ["标题1", "标题2"],
  "suggestedTags": ["标签1", "标签2"]
}
```

### 代码审查模板

```
你是一位资深的代码审查专家。

编程语言：{language}
代码内容：
{code}

请审查代码并提供建议，输出JSON格式：
{
  "summary": "总体评价",
  "issues": ["问题列表"],
  "suggestions": ["改进建议"]
}
```

### 翻译模板

```
请将以下{sourceLanguage}文本翻译成{targetLanguage}。

原文：
{originalText}

要求：
1. 保持原文语气和风格
2. 确保翻译准确流畅
3. 适当本地化表达

请直接输出翻译结果。
```

## 故障排查

### 问题1：模板未找到

**错误信息**：`未找到内容优化模板，请先初始化数据库`

**解决方案**：
1. 检查数据库连接是否正常
2. 确认模板表是否存在
3. 运行初始化 SQL 脚本

### 问题2：缓存未更新

**现象**：修改模板后，使用的还是旧版本

**解决方案**：
1. 检查缓存配置是否正确
2. 使用 DELETE 操作会自动清空缓存
3. 重启应用强制刷新缓存

### 问题3：变量替换失败

**现象**：输出中仍包含 `{variableName}` 占位符

**解决方案**：
1. 检查变量名是否拼写正确
2. 确认传入的变量 Map 包含所有必需变量
3. 查看日志确认变量值

## 监控和维护

### 查看模板使用情况

```sql
-- 查看所有启用的模板
SELECT template_name, category, description, updated_at 
FROM prompt_template 
WHERE enabled = TRUE;

-- 查看最近更新的模板
SELECT template_name, updated_at, updated_by 
FROM prompt_template 
ORDER BY updated_at DESC 
LIMIT 10;
```

### 备份模板

```bash
mysqldump -u root -p llm_platform prompt_template > prompt_template_backup.sql
```

### 恢复模板

```bash
mysql -u root -p llm_platform < prompt_template_backup.sql
```

