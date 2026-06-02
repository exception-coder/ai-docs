# feature-config 编码摘要

> 配套：`feature-config-通用配置存储-current.md` + `-api-current.md`
> 最后更新：2026-05-25

## 1. 设计结论 / 核心业务规则（摘要）

- 单表 KV：`feature_config(feature_id PK TEXT, value_json TEXT NOT NULL, updated_at INTEGER NOT NULL)`
- 后端不校验 `value_json` 内部结构，只校验是合法 JSON 且非 null
- `featureId` 正则 `^[a-z][a-z0-9-]{0,63}$`
- `GET` 不存在返回 **404**（不是空对象，让前端区分"未配置"）
- `PUT` 是 upsert；`DELETE` 幂等返回 204
- 前端 `useFeatureConfig` 浅合并 `defaults ⊕ remote`；遇 404 时尝试 localStorage 迁移
- 不加密、不审计、不分用户

## 2. 接口入口指针

| 方法 | 路径 | 实现 |
|---|---|---|
| GET | `/api/feature-configs/{featureId}` | `com.exceptioncoder.toolbox.common.featureconfig.api.FeatureConfigController#get` |
| PUT | `/api/feature-configs/{featureId}` | `com.exceptioncoder.toolbox.common.featureconfig.api.FeatureConfigController#save` |
| DELETE | `/api/feature-configs/{featureId}` | `com.exceptioncoder.toolbox.common.featureconfig.api.FeatureConfigController#delete` |

字段级契约见 `feature-config-通用配置存储-api-current.md`。

## 3. 涉及类清单

### 3.1 后端（新增）

**`com.exceptioncoder.toolbox.common.featureconfig.domain.FeatureConfig`**
- POJO（lombok `@Data @Builder`）：`String featureId; String valueJson; long updatedAt;`

**`com.exceptioncoder.toolbox.common.featureconfig.repository.FeatureConfigRepository`**
- 依赖 `JdbcTemplate`
- 方法：
  - `Optional<FeatureConfig> findById(String featureId)`
  - `void upsert(FeatureConfig cfg)` → `INSERT INTO feature_config(...) VALUES(?,?,?) ON CONFLICT(feature_id) DO UPDATE SET value_json=excluded.value_json, updated_at=excluded.updated_at`
  - `void deleteById(String featureId)`
- ROW mapper 参考 `HostRepository.ROW` 写法

**`com.exceptioncoder.toolbox.common.featureconfig.service.FeatureConfigService`**
- 依赖 `FeatureConfigRepository` + `ObjectMapper`
- `FeatureConfig findRequired(String featureId)`：找不到抛 `NoSuchElementException`（被 GlobalExceptionHandler 转 404）
  - 若现有 handler 无此映射，则改抛自定义 `FeatureConfigNotFoundException`（继承 `RuntimeException`），在 controller 层 `@ExceptionHandler` 转 404 —— **施工前确认 handler 现状**
- `FeatureConfig save(String featureId, JsonNode value)`：
  - 校验 `featureId` 正则
  - 校验 `value != null && !value.isNull()`
  - `valueJson = objectMapper.writeValueAsString(value)`
  - `updatedAt = System.currentTimeMillis()`
  - 调用 `repository.upsert(...)`
- `void delete(String featureId)`：直接 `repository.deleteById`
- featureId 正则在 service 层做（不放 controller bean validation，方便单元测试）

**`com.exceptioncoder.toolbox.common.featureconfig.api.FeatureConfigController`**
- `@RestController @RequestMapping("/api/feature-configs")`
- 注入 `FeatureConfigService` + `ObjectMapper`
- `GET /{featureId}` → `FeatureConfigView.from(service.findRequired(id), objectMapper)`
- `PUT /{featureId}` body=`FeatureConfigSaveRequest` → `service.save(id, req.value())`，转 view
- `DELETE /{featureId}` → 204
- 异常映射：`featureId 不合法` → 抛 `IllegalArgumentException`（已被 GlobalExceptionHandler 转 400）

**`com.exceptioncoder.toolbox.common.featureconfig.api.dto.FeatureConfigSaveRequest`**
- `record FeatureConfigSaveRequest(JsonNode value)`
- `@NotNull` value（Jackson 反序列化 null 会得到 `NullNode`，由 service 内做 `isNull()` 校验，注解只防 missing field）

**`com.exceptioncoder.toolbox.common.featureconfig.api.dto.FeatureConfigView`**
- `record FeatureConfigView(String featureId, JsonNode value, long updatedAt)`
- 静态工厂 `from(FeatureConfig cfg, ObjectMapper om)`：`om.readTree(cfg.getValueJson())`

### 3.2 前端（新增）

**`frontend/src/lib/featureConfig.ts`**

类型：
```ts
export interface FeatureConfigView<T> {
  featureId: string
  value: T
  updatedAt: number
}
export interface UseFeatureConfigOptions<T> {
  defaults: T
  /** 设了则在库无记录时尝试从 localStorage 该 key 读取旧值并迁移到库 */
  legacyLocalStorageKey?: string
  /** 调用方决定怎么合并 defaults 与远端 value，默认浅合并 */
  merge?: (defaults: T, remote: Partial<T>) => T
}
```

函数：
- `getFeatureConfig<T>(featureId): Promise<FeatureConfigView<T> | null>`（404 → null，不抛）
- `putFeatureConfig<T>(featureId, value): Promise<FeatureConfigView<T>>`
- `deleteFeatureConfig(featureId): Promise<void>`
- `useFeatureConfig<T>(featureId, opts): { config, isReady, isLoading, error, setConfig, resetConfig }`
  - 内部用 TanStack Query：`useQuery({ queryKey: ['feature-config', featureId], queryFn })`
  - `queryFn`：先 GET，404 时若有 `legacyLocalStorageKey` 则尝试迁移（try parse → PUT → removeItem），任一步骤失败回退 defaults
  - `setConfig`：`useMutation` PUT，成功后 `queryClient.setQueryData`
  - `resetConfig`：`useMutation` DELETE，成功后 `queryClient.setQueryData(..., null)`
  - `isReady`：query 状态非 `pending`

**`frontend/src/features/java8gu/data.ts`**
- 不再读写 localStorage；删除 `CONFIG_KEY` 直接读写
- `getDataSource()` 和 `setDataSource()` 改为：
  - 选项 A（推荐）：**删除这两个函数**，所有调用方改用 `useFeatureConfig`
  - 选项 B（兼容）：保留 `getDataSource()` 但改为 `async` 走 `getFeatureConfig`，`setDataSource()` 改为 `async` 走 `putFeatureConfig` → 注意现有 `cacheKeyOf(cfg)` 同步使用 cfg，需要先 `await getDataSource()` 再调用
  - **决策：先做选项 B**，因为 `loadIndex`/`loadMarkdown`/`resetRuntimeState` 都依赖同步 `getDataSource`；改完后再评估是否值得做选项 A 重构
- `DEFAULT_DATA_SOURCE` 保留为前端默认值；`legacyLocalStorageKey = 'java8gu:source:v1'`
- **缓存 key**：本来用 owner/repo/branch/dir 组合做 localStorage 缓存 key，迁移后 cfg 取值变异步 → 需要先 `await` 拿到 cfg 再调用 `loadIndex`。调用栈梳理：
  - `loadIndex()` → `doLoadIndex()` → `getDataSource()` 处改 `await getDataSource()`，函数本身已 async
  - `loadMarkdown(id)` → 内部对 `getDataSource()` 的调用 → 加 await
  - `resetRuntimeState()` 不读 cfg，无影响
- 索引缓存仍写 localStorage（这是大块 JSON 缓存，不属于 feature config 范畴，不要塞到 DB）

**`frontend/src/features/java8gu/components/DataSourceDialog.tsx`**
- 不再调 `setDataSource` 同步函数
- 改为接收一个 `onSaved: (cfg) => Promise<void>` 由父组件用 `setConfig` 实现
- `handleSave` 改 async + try/catch；保存中按钮加 loading
- 或更彻底：弹层内直接 `const { setConfig } = useFeatureConfig('java8gu', ...)`；父组件只控制 open 状态

**`frontend/src/features/java8gu/pages/Java8guHubPage.tsx`（含 Category / Question 页）**
- 顶部用 `useFeatureConfig<DataSourceConfig>('java8gu', { defaults: DEFAULT_DATA_SOURCE, legacyLocalStorageKey: 'java8gu:source:v1' })`
- `isReady=false` 期间显示 skeleton / loading 文本（不阻塞 sidebar）
- 把 `config` 当依赖传给 `loadIndex`（或先 `await` 后再调用）

## 4. 数据结构

### 4.1 SQL

```sql
-- toolbox-common/src/main/resources/db/feature-config-schema.sql
CREATE TABLE IF NOT EXISTS feature_config (
    feature_id  TEXT PRIMARY KEY,
    value_json  TEXT NOT NULL,
    updated_at  INTEGER NOT NULL
);
```

不需要额外索引（PK 已是唯一索引）。

### 4.2 后端 ↔ JSON 序列化

- `value_json` 列存的是 Jackson `writeValueAsString(JsonNode)` 后的字符串
- 读时 `objectMapper.readTree(string)` 还原为 `JsonNode`，直接交给 Spring 序列化输出
- **不**用 `String value` 字段直接对外，避免输出时被二次转义成字符串

### 4.3 前端类型

```ts
// java8gu 的 value 结构由该 feature 自己定义
// 这里 DataSourceConfig 来自 features/java8gu/data.ts，不放到通用 lib
```

## 5. 重要约束与边界

- 表迁移：`SchemaInitializer` 启动期跑 `classpath*:db/*-schema.sql`，本表 SQL 必须 `IF NOT EXISTS`（CLAUDE.md §48 强约束）
- 事务：upsert 单语句；不需要显式 `@Transactional`
- 并发：单用户本地，不加乐观锁；后写覆盖先写
- localStorage 迁移**只清 `legacyLocalStorageKey` 指定的 key**，不动其他 java8gu 缓存（如索引缓存 `java8gu:index:v1:*`）
- mock 模式（`lib/mock/`）暂不为 `/api/feature-configs/*` 注册 mock handler；mock 关闭时按真实路径走即可。如未来需要，再在 `lib/mock/registry` 加上
- 不在 controller 上挂 `@Validated` + bean validation 校验 featureId 正则（用 service 内显式校验，前端也已限制）→ 简化依赖
- toolbox-starter 已 `componentScan` 覆盖 `com.exceptioncoder.toolbox.common.*`（参考其他 common 组件如 `SseEmitterRegistry` 直接被注入）；新 `featureconfig` 子包零额外配置即可被扫到 —— **施工时第一步先确认 `@SpringBootApplication` 的 scanBasePackages**

## 6. 验证 / 测试要点

- `mvn -pl toolbox-common -am test` 通过；如新增单测覆盖 service：正则校验、null value、upsert、delete 幂等
- 启动后 `~/.kai-toolbox/toolbox.db` 中可见 `feature_config` 表
- curl 三件套：PUT → GET 回读 → DELETE → GET 拿 404
- java8gu 端到端：
  - 清空 DB 行 + 浏览器 localStorage 留 `java8gu:source:v1` 值 → 进入页面 → DB 出现行 + localStorage 该 key 被清
  - 修改弹层保存 → 换浏览器 :8080 看到改后值
  - 点重置默认按钮（弹层）+ 保存 → DB 内 value 还原为 defaults（注意：弹层的"重置默认"是改 draft，不是 DELETE；如果要做真正的"删除配置回落 defaults"，需要 UI 上加一个独立的按钮调 `resetConfig`）
- 后端 down：进入 java8gu → toast 报错 + 回落 DEFAULT，仍能读题
