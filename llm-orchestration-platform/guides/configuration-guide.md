# 配置文件说明

## 配置文件结构

项目采用**按环境分目录 + 按类型分文件**的配置结构：

```
llm-starter/src/main/resources/
├── application.yml              # 主配置文件（动态导入环境配置）
├── application-llm.yml          # LLM 业务配置（所有环境通用）
└── config/
    ├── dev/                     # 开发环境配置目录
    │   ├── datasource.yml           # 数据源配置
    │   ├── spring-ai.yml            # Spring AI 配置
    │   └── logging.yml              # 日志配置
    ├── test/                    # 测试环境配置目录
    │   ├── datasource.yml
    │   ├── spring-ai.yml
    │   └── logging.yml
    └── prod/                    # 生产环境配置目录
        ├── datasource.yml
        ├── spring-ai.yml
        └── logging.yml
```

## 配置加载机制

### 动态导入配置

`application.yml` 使用 Spring Boot 的动态配置导入功能：

```yaml
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  
  config:
    import:
      - classpath:application-llm.yml
      - optional:classpath:config/${spring.profiles.active}/datasource.yml
      - optional:classpath:config/${spring.profiles.active}/spring-ai.yml
      - optional:classpath:config/${spring.profiles.active}/logging.yml
```

**工作原理：**
- `${spring.profiles.active}` 会被替换为实际的环境名（dev/test/prod）
- `optional:` 前缀表示文件不存在时不报错
- Spring Boot 会根据激活的 profile 自动加载对应目录下的配置文件

**示例：**
```bash
# 当激活 dev 环境时，会加载：
- application.yml
- application-llm.yml
- config/dev/datasource.yml
- config/dev/spring-ai.yml
- config/dev/logging.yml

# 当激活 prod 环境时，会加载：
- application.yml
- application-llm.yml
- config/prod/datasource.yml
- config/prod/spring-ai.yml
- config/prod/logging.yml
```

## 配置文件说明

### 1. application.yml（主配置）
- 服务器端口：8080
- 应用名称：llm-orchestration-platform
- **默认激活环境：dev**
- **动态导入环境特定配置**
- Actuator 监控配置

### 2. application-llm.yml（LLM 业务配置 - 通用）
- LLM 默认提供商和模型
- OpenAI/Ollama/DeepSeek 业务配置
- 通用配置（超时、重试等）
- **所有环境共用**

### 3. 环境特定配置（按目录组织）

每个环境目录包含 3 个配置文件：

| 配置文件 | 说明 | 包含内容 |
|---------|------|---------|
| datasource.yml | 数据源配置 | DataSource + Hikari + JPA |
| spring-ai.yml | Spring AI 配置 | OpenAI + Ollama + Qdrant |
| logging.yml | 日志配置 | 日志级别 + 格式 + 文件 |

## 环境配置对比

| 配置项 | dev | test | prod |
|--------|-----|------|------|
| **数据库** |
| 数据库名 | llm_platform_dev | llm_platform_test | 环境变量 |
| ddl-auto | update | validate | none |
| show-sql | true | false | false |
| 连接池大小 | 5 | 10 | 20 |
| **向量库** |
| Collection | job_postings_dev | job_postings_test | job_postings |
| Qdrant TLS | false | true | true |
| 自动初始化 | true | true | false |
| **日志** |
| 应用日志级别 | DEBUG | INFO | WARN |
| SQL 日志 | TRACE | INFO | WARN |
| 日志路径 | logs/dev/ | logs/test/ | /var/log/ |
| 保留天数 | 7 | 30 | 90 |
| **LLM** |
| 默认模型 | gpt-3.5-turbo | gpt-3.5-turbo | gpt-4 |
| Temperature | 0.7 | 0.5 | 0.3 |

## 环境切换

### 方式 1：命令行参数
```bash
# 开发环境（默认）
java -jar llm-platform.jar

# 测试环境
java -jar llm-platform.jar --spring.profiles.active=test

# 生产环境
java -jar llm-platform.jar --spring.profiles.active=prod
```

### 方式 2：环境变量
```bash
export SPRING_PROFILES_ACTIVE=prod
java -jar llm-platform.jar
```

### 方式 3：IDE 配置
在 IDEA 的 Run Configuration 中设置：
- Environment variables: `SPRING_PROFILES_ACTIVE=dev`
- 或 Program arguments: `--spring.profiles.active=dev`

## 配置文件优势

### ✅ 结构清晰
- 按环境分目录：每个环境的配置独立管理
- 按类型分文件：数据源、AI、日志配置分离
- 职责单一：每个文件只负责一类配置

### ✅ 易于维护
- 修改某个环境的某类配置，直接定位到对应文件
- 环境配置互不影响
- 便于代码审查和版本控制

### ✅ 扩展性好
- 添加新环境：创建新目录，复制 3 个配置文件
- 添加新配置类型：在每个环境目录下添加新文件，在 application.yml 中导入
- 支持按需加载配置

### ✅ 动态灵活
- 使用 `${spring.profiles.active}` 动态引用
- 无需为每个环境创建 `application-{profile}.yml`
- 配置文件自动根据环境加载

## 添加新环境

假设要添加 `staging` 环境：

1. 创建目录：
```bash
mkdir -p src/main/resources/config/staging
```

2. 复制配置文件：
```bash
cp config/test/*.yml config/staging/
```

3. 修改配置值（无需修改 application.yml）

4. 启动时指定环境：
```bash
java -jar app.jar --spring.profiles.active=staging
```

## 添加新配置类型

假设要添加缓存配置：

1. 在每个环境目录下创建 `cache.yml`：
```
config/dev/cache.yml
config/test/cache.yml
config/prod/cache.yml
```

2. 在 `application.yml` 中添加导入：
```yaml
spring:
  config:
    import:
      - optional:classpath:config/${spring.profiles.active}/cache.yml
```

## 配置优先级

Spring Boot 配置优先级（从高到低）：
1. 命令行参数
2. 环境变量
3. `config/{profile}/logging.yml`
4. `config/{profile}/spring-ai.yml`
5. `config/{profile}/datasource.yml`
6. `application-llm.yml`
7. `application.yml`

## 最佳实践

### 开发环境
- 使用本地服务（MySQL、Qdrant、Ollama）
- 启用详细日志便于调试
- 自动更新数据库结构
- 配置文件位置：`config/dev/`

### 测试环境
- 使用独立的测试服务器
- 模拟生产环境配置
- 只验证数据库结构，不自动修改
- 配置文件位置：`config/test/`

### 生产环境
- **所有敏感信息必须通过环境变量提供**
- 禁止自动修改数据库结构
- 使用 WARN 级别日志
- 启用 TLS 加密
- 启用连接泄漏检测
- 配置文件位置：`config/prod/`

## 故障排查

### 配置未生效
1. 检查 `spring.profiles.active` 是否正确设置
2. 确认配置文件路径：`config/{profile}/{type}.yml`
3. 查看启动日志中的 "Loaded config file"

### 找不到配置文件
确保配置文件在正确的位置：
```
src/main/resources/config/dev/datasource.yml
src/main/resources/config/dev/spring-ai.yml
src/main/resources/config/dev/logging.yml
```

### 环境变量未生效
1. 确认环境变量名称正确（大写，下划线分隔）
2. 检查是否在启动前设置了环境变量
3. 使用 `${VAR_NAME}` 而不是 `${VAR_NAME:default}` 强制要求环境变量

## 配置加密

生产环境建议使用：
- **Spring Cloud Config Server**：集中管理配置
- **HashiCorp Vault**：密钥管理
- **AWS Secrets Manager**：AWS 环境
- **Azure Key Vault**：Azure 环境
- **Kubernetes Secrets**：K8s 环境
