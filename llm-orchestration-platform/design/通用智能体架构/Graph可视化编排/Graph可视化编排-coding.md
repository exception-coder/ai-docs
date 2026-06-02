# 编码摘要文档

> 本文档由完整设计文档精简而来，供 AI 辅助编码时使用，聚焦实现所需的最小必要信息。
> 对应完整文档：`Graph可视化编排-20260412-v1.md`
>
> **职责边界**：设计文档回答"哪些类、什么职责、怎么协作"，本文档回答"每个方法怎么写"。

---

## 变更记录

| 版本 | 日期 | 变更内容摘要 |
|------|------|--------------|
| v1 | 2026-04-12 | 初始版本 |

---

## 1. 核心业务规则

- 规则1：GraphNode 中 `type === 'LLM'` 且 `config.agentId` 非空的节点使用 AgentNode 自定义组件渲染；其他类型节点使用 Vue Flow 默认节点
- 规则2：GraphEdge 有 `condition` 字段的使用 ConditionalEdge 自定义组件；无 condition 的使用默认动画边
- 规则3：`entryNodeId` 匹配的节点添加入口标记样式（主题色边框 + 入口图标）
- 规则4：Dagre 布局方向 TB，节点宽 200px 高 100px，nodesep 80，ranksep 120
- 规则5：点击 LLM 节点（有 agentId）打开 ToolPanel；点击其他节点仅高亮，不打开面板
- 规则6：Tool 列表在页面初始化时批量加载到 toolsMap，加载失败静默跳过
- 规则7：Graph 数据加载失败显示空画布 + ElMessage.error；Agent 加载失败节点显示"未知"

---

## 2. 接口契约

### 复用接口（无新增）

```
GET /api/v1/graphs/{graphId}
返回：GraphDefinition - { id, name, description, nodes[], edges[], entryNodeId }

GET /api/v1/graphs/{graphId}/agents
返回：List<AgentDefinition> - [{ id, name, description, toolIds[], llmProvider, llmModel, ... }]

GET /api/v1/agents/{agentId}/tools
返回：List<ToolDefinition> - [{ id, name, description, inputSchema, type, isAsync }]
```

### 前端已有 API 调用（`src/api/index.js`，不修改）

```js
graphAPI.getById(id)        // GET /graphs/{id}
graphAPI.getAgents(id)      // GET /graphs/{id}/agents
agentAPI.getTools(id)       // GET /agents/{id}/tools
```

---

## 3. 涉及类清单（全路径）

| 全路径 | 操作 | 说明 |
|--------|------|------|
| `src/views/GraphVisualization.vue` | 新建 | 可视化页面主视图 |
| `src/components/graph/AgentNode.vue` | 新建 | 自定义 Agent 节点组件 |
| `src/components/graph/ConditionalEdge.vue` | 新建 | 自定义条件边组件 |
| `src/components/graph/ToolPanel.vue` | 新建 | Agent Tool 侧边抽屉 |
| `src/composables/useGraphLayout.js` | 新建 | Dagre 自动布局 composable |
| `src/views/GraphOrchestration.vue` | 改造 | 新增 VISUAL 按钮 + router-link |
| `src/router/index.js` | 改造 | 新增路由条目 |

### 关键方法签名与职责

#### `src/composables/useGraphLayout.js`

```js
/**
 * useGraphLayout(nodes, edges, options?) — Dagre 自动布局
 * @param {Array} nodes - Vue Flow nodes（无 position 或 position 为 {0,0}）
 * @param {Array} edges - Vue Flow edges
 * @param {Object} options - { direction: 'TB', nodeWidth: 200, nodeHeight: 100, nodesep: 80, ranksep: 120 }
 * @returns {Array} nodes — 带计算后 position 的 nodes 副本
 */
export function useGraphLayout(nodes, edges, options = {}) {
  // 1. 创建 dagre Graph 实例，setGraph({ rankdir, nodesep, ranksep })
  // 2. 遍历 nodes，setNode(id, { width, height })
  // 3. 遍历 edges，setEdge(source, target)
  // 4. dagre.layout(g)
  // 5. 遍历 nodes，从 g.node(id) 取 x/y，计算 position = { x: x - width/2, y: y - height/2 }
  // 6. 返回带 position 的新 nodes 数组
}
```

#### `src/views/GraphVisualization.vue`

```js
// 核心 reactive 状态
const nodes = ref([])               // Vue Flow nodes
const edges = ref([])               // Vue Flow edges
const selectedAgent = ref(null)     // 当前选中的 Agent（含 tools）
const loading = ref(false)
const graphData = ref(null)         // 原始 Graph 数据

// nodeTypes 注册
const nodeTypes = { agentNode: AgentNode }
const edgeTypes = { conditionalEdge: ConditionalEdge }

/**
 * loadGraphData(graphId) — 页面初始化数据加载
 * 1. 并行调用 graphAPI.getById(graphId) + graphAPI.getAgents(graphId)
 * 2. 构建 agentMap: { [agentId]: AgentDefinition }
 * 3. 为每个 agent 调用 agentAPI.getTools(agent.id)，构建 toolsMap: { [agentId]: ToolDefinition[] }
 * 4. 调用 mapToVueFlow(graph, agentMap, toolsMap) 生成 nodes/edges
 * 5. 调用 useGraphLayout(nodes, edges) 计算位置
 * 6. 赋值给 reactive 状态
 */

/**
 * mapToVueFlow(graph, agentMap, toolsMap) — 数据映射
 *
 * nodes 映射规则：
 *   graph.nodes.map(node => ({
 *     id: node.id,
 *     type: (node.type === 'LLM' && node.config?.agentId) ? 'agentNode' : 'default',
 *     position: { x: 0, y: 0 },  // 后续由 dagre 填充
 *     data: {
 *       label: node.name || node.id,
 *       nodeType: node.type,
 *       isEntry: node.id === graph.entryNodeId,
 *       agentId: node.config?.agentId || null,
 *       agent: agentMap[node.config?.agentId] || null,
 *       tools: toolsMap[node.config?.agentId] || [],
 *     }
 *   }))
 *
 * edges 映射规则：
 *   graph.edges.map(edge => ({
 *     id: `${edge.from}-${edge.to}`,
 *     source: edge.from,
 *     target: edge.to,
 *     type: edge.condition ? 'conditionalEdge' : 'default',
 *     label: edge.condition || '',
 *     animated: !edge.condition,  // 无条件边动画，条件边静态
 *     data: { condition: edge.condition || '' },
 *   }))
 */

/**
 * onNodeClick(event, node) — 节点点击事件
 * 1. 如果 node.data.agentId 存在 → selectedAgent.value = { ...node.data.agent, tools: node.data.tools }
 * 2. 否则 → selectedAgent.value = null
 */
```

#### `src/components/graph/AgentNode.vue`

```vue
<!--
Props（通过 Vue Flow 注入）:
  - data: { label, nodeType, isEntry, agentId, agent, tools }

结构:
  - 外层 div: neo-convex 圆角卡片，isEntry 时添加主题色边框
  - 顶部: 图标(Bot/Cpu/GitBranch 按 nodeType) + label + nodeType 标签
  - 中部: agent.name（如有）+ llmModel 标签
  - 底部: tools 标签列表（最多显示 3 个，超出显示 +N）
  - Handle: top(target) + bottom(source)

Emits:
  - 无自定义 emit，点击通过 Vue Flow 的 @nodeClick 捕获
-->
```

#### `src/components/graph/ConditionalEdge.vue`

```vue
<!--
Props（通过 Vue Flow 注入）:
  - id, sourceX, sourceY, targetX, targetY, data: { condition }

结构:
  - 使用 Vue Flow 的 getBezierPath 计算路径
  - 渲染 <path> SVG 元素（虚线样式 stroke-dasharray）
  - 在路径中点渲染 condition 文本标签（foreignObject）
  - 标签样式: neo-concave 小圆角背景 + 小号字体
-->
```

#### `src/components/graph/ToolPanel.vue`

```vue
<!--
Props:
  - agent: { id, name, description, llmProvider, llmModel, tools: ToolDefinition[] }
  - visible: boolean

Emits:
  - update:visible (关闭抽屉)

结构:
  - el-drawer 右侧抽屉，宽度 400px
  - Header: Agent 名称 + 描述 + llmProvider/llmModel 标签
  - Body: Tool 列表
    - 每个 Tool: neo-concave 卡片
      - Tool name + type 标签（FUNCTION/RETRIEVER/...）
      - description 文本
      - inputSchema 折叠展示（JSON 高亮，复用 highlight.js）
  - Footer: 无
-->
```

#### `src/views/GraphOrchestration.vue`（改造）

```js
// 新增 import
import { useRouter } from 'vue-router'
const router = useRouter()

// 新增方法
const viewVisual = (graph) => {
  router.push({ name: 'GraphVisualization', params: { id: graph.id } })
}

// 模板中每个 graph 卡片的按钮区域新增:
// <button @click="viewVisual(graph)">VISUAL</button>
// 放在 CHAIN 按钮之前
```

#### `src/router/index.js`（改造）

```js
// 新增路由条目
{
  path: '/graph/:id/visual',
  name: 'GraphVisualization',
  component: () => import('@/views/GraphVisualization.vue'),
  meta: { title: 'Graph 可视化' }
}
```

---

## 4. 数据结构

### Vue Flow Node 结构

```js
{
  id: 'scan-node',                    // GraphNode.id
  type: 'agentNode',                  // 'agentNode' | 'default'
  position: { x: 100, y: 50 },       // dagre 计算
  data: {
    label: '代码扫描',                 // GraphNode.name
    nodeType: 'LLM',                  // GraphNode.type
    isEntry: true,                    // node.id === graph.entryNodeId
    agentId: 'code-awareness-agent',  // GraphNode.config.agentId
    agent: {                          // AgentDefinition（可能为 null）
      id: 'code-awareness-agent',
      name: '代码感知智能体',
      llmProvider: 'dashscope',
      llmModel: 'qwen-max',
      // ...
    },
    tools: [                          // ToolDefinition[]（可能为空数组）
      { id: 'file-tree-tool', name: '文件树提取工具', type: 'FUNCTION', ... }
    ]
  }
}
```

### Vue Flow Edge 结构

```js
{
  id: 'review-node-design-node',       // `${from}-${to}`
  source: 'review-node',               // GraphEdge.from
  target: 'design-node',               // GraphEdge.to
  type: 'conditionalEdge',             // 有 condition 时
  label: '审查不通过',                  // GraphEdge.condition
  animated: false,                      // 条件边不动画
  data: { condition: '审查不通过' }
}
```

---

## 5. 重要约束与边界

- **Vue Flow 样式隔离**：必须在 GraphVisualization.vue 中导入 `@vue-flow/core/dist/style.css` 和 `@vue-flow/core/dist/theme-default.css`，否则节点/边不渲染
- **Dagre 坐标原点**：dagre 输出的 x/y 是节点中心点坐标，需减去 width/2 和 height/2 转为左上角坐标（Vue Flow 使用左上角定位）
- **Handle 组件导入**：AgentNode.vue 必须从 `@vue-flow/core` 导入 `Handle` 和 `Position`，否则连线无法连接到节点
- **nodeTypes 注册**：自定义节点类型必须在 VueFlow 组件的 `:node-types` prop 中注册，key 必须与 node.type 一致
- **不处理的场景**：PARALLEL 节点的并行分支可视化（P2）；LOOP 节点的循环标记（P2）；运行态实时更新（P3）

---

## 6. 下游依赖调用

```
// 前端 API 调用（src/api/index.js 已有，不修改）
graphAPI.getById(graphId)       → GET /api/v1/graphs/{graphId}
graphAPI.getAgents(graphId)     → GET /api/v1/graphs/{graphId}/agents
agentAPI.getTools(agentId)      → GET /api/v1/agents/{agentId}/tools
```

---

## 7. 异常处理要点

- `graphAPI.getById` 失败 → ElMessage.error("Graph 数据加载失败") + 显示空画布 + 重试按钮
- `graphAPI.getAgents` 失败 → console.error + 节点正常渲染，Agent 信息显示为空
- `agentAPI.getTools(agentId)` 失败 → console.error + 该 Agent 的 tools 置为空数组，静默跳过
- 路由参数 `id` 缺失 → router.replace('/graph')（重定向回列表页）

---

## 8. npm 依赖安装

```bash
cd llm-frontend
npm install @vue-flow/core @vue-flow/background @vue-flow/controls @vue-flow/minimap @dagrejs/dagre
```

---

## 9. 文件创建顺序

1. `npm install` 安装依赖
2. `src/composables/useGraphLayout.js` — 无外部组件依赖，可独立完成
3. `src/components/graph/AgentNode.vue` — 依赖 @vue-flow/core 的 Handle/Position
4. `src/components/graph/ConditionalEdge.vue` — 依赖 @vue-flow/core 的 getBezierPath
5. `src/components/graph/ToolPanel.vue` — 依赖 Element Plus el-drawer
6. `src/views/GraphVisualization.vue` — 依赖以上所有
7. `src/router/index.js` — 新增路由条目
8. `src/views/GraphOrchestration.vue` — 新增 VISUAL 按钮
