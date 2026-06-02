# Prompt 测试功能使用指南

## 功能概述

Prompt 测试功能允许你在前端页面验证不同模板和模型的输出效果，方便调试和优化 Prompt。

## 核心功能

1. **选择模板**：从数据库中选择已有的 Prompt 模板
2. **填写变量**：根据模板要求填写变量值
3. **选择模型**：选择要测试的 LLM 模型（OpenAI、Ollama 等）
4. **查看结果**：实时查看渲染后的 Prompt 和 LLM 输出结果
5. **性能监控**：显示 Token 使用情况和执行耗时

## API 接口

### 1. 执行 Prompt 测试

**接口**：`POST /api/v1/prompt-test`

**请求示例**：

```bash
curl -X POST http://localhost:8080/api/v1/prompt-test \
  -H "Content-Type: application/json" \
  -d '{
    "templateName": "content-optimization",
    "variables": {
      "platformCharacteristics": "年轻女性为主，注重生活方式和美学",
      "styleDescription": "轻松随意，口语化表达",
      "originalContent": "今天试用了一款新的护肤品，效果还不错。",
      "contentType": "产品测评",
      "targetAudience": "18-35岁女性",
      "additionalRequirements": "无"
    },
    "model": "gpt-3.5-turbo",
    "temperature": 0.8,
    "maxTokens": 1000
  }'
```

**响应示例**：

```json
{
  "renderedPrompt": "你是一位专业的内容创作优化专家...",
  "output": "姐妹们！今天要给你们分享一个宝藏护肤品💎...",
  "model": "gpt-3.5-turbo",
  "provider": "openai",
  "tokenUsage": {
    "promptTokens": 450,
    "completionTokens": 280,
    "totalTokens": 730
  },
  "executionTime": 2350
}
```

### 2. 获取可用模型列表

**接口**：`GET /api/v1/prompt-test/models`

**请求示例**：

```bash
curl http://localhost:8080/api/v1/prompt-test/models
```

**响应示例**：

```json
[
  {
    "code": "gpt-4",
    "provider": "openai",
    "name": "GPT-4",
    "description": "最强大的模型，适合复杂任务"
  },
  {
    "code": "gpt-3.5-turbo",
    "provider": "openai",
    "name": "GPT-3.5 Turbo",
    "description": "性价比高，适合大多数场景"
  },
  {
    "code": "llama2",
    "provider": "ollama",
    "name": "Llama 2",
    "description": "Meta 开源模型"
  }
]
```

### 3. 获取模板变量示例

**接口**：`GET /api/v1/prompt-test/template-variables/{templateName}`

**请求示例**：

```bash
curl http://localhost:8080/api/v1/prompt-test/template-variables/content-optimization
```

**响应示例**：

```json
{
  "platformCharacteristics": "年轻女性为主，注重生活方式和美学",
  "styleDescription": "轻松随意，口语化表达",
  "originalContent": "今天试用了一款新的护肤品，效果还不错。",
  "contentType": "产品测评",
  "targetAudience": "18-35岁女性",
  "additionalRequirements": "无"
}
```

## 前端集成示例

### Vue 3 示例

```vue
<template>
  <div class="prompt-test">
    <h2>Prompt 测试工具</h2>
    
    <!-- 模板选择 -->
    <div class="form-group">
      <label>选择模板：</label>
      <select v-model="form.templateName" @change="loadTemplateVariables">
        <option value="content-optimization">内容优化</option>
        <option value="simple-chat">简单对话</option>
        <option value="code-review">代码审查</option>
      </select>
    </div>
    
    <!-- 模型选择 -->
    <div class="form-group">
      <label>选择模型：</label>
      <select v-model="form.model">
        <option v-for="model in models" :key="model.code" :value="model.code">
          {{ model.name }} - {{ model.description }}
        </option>
      </select>
    </div>
    
    <!-- 变量输入 -->
    <div class="form-group">
      <label>模板变量：</label>
      <div v-for="(value, key) in form.variables" :key="key" class="variable-input">
        <label>{{ key }}:</label>
        <textarea v-model="form.variables[key]" rows="3"></textarea>
      </div>
    </div>
    
    <!-- 高级选项 -->
    <div class="form-group">
      <label>温度参数 (0-1)：</label>
      <input v-model.number="form.temperature" type="number" step="0.1" min="0" max="1">
    </div>
    
    <div class="form-group">
      <label>最大 Token 数：</label>
      <input v-model.number="form.maxTokens" type="number" min="100" max="4000">
    </div>
    
    <!-- 提交按钮 -->
    <button @click="testPrompt" :disabled="loading">
      {{ loading ? '测试中...' : '开始测试' }}
    </button>
    
    <!-- 结果展示 -->
    <div v-if="result" class="result">
      <h3>测试结果</h3>
      
      <div class="result-section">
        <h4>渲染后的 Prompt：</h4>
        <pre>{{ result.renderedPrompt }}</pre>
      </div>
      
      <div class="result-section">
        <h4>LLM 输出：</h4>
        <pre>{{ result.output }}</pre>
      </div>
      
      <div class="result-section">
        <h4>性能指标：</h4>
        <ul>
          <li>使用模型：{{ result.model }}</li>
          <li>执行耗时：{{ result.executionTime }}ms</li>
          <li>Prompt Tokens：{{ result.tokenUsage?.promptTokens }}</li>
          <li>Completion Tokens：{{ result.tokenUsage?.completionTokens }}</li>
          <li>总 Tokens：{{ result.tokenUsage?.totalTokens }}</li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import axios from 'axios';

const form = ref({
  templateName: 'content-optimization',
  model: 'gpt-3.5-turbo',
  variables: {},
  temperature: 0.7,
  maxTokens: 2000
});

const models = ref([]);
const result = ref(null);
const loading = ref(false);

// 加载可用模型
onMounted(async () => {
  const response = await axios.get('/api/v1/prompt-test/models');
  models.value = response.data;
  
  // 加载默认模板变量
  loadTemplateVariables();
});

// 加载模板变量示例
async function loadTemplateVariables() {
  const response = await axios.get(
    `/api/v1/prompt-test/template-variables/${form.value.templateName}`
  );
  form.value.variables = response.data;
}

// 执行测试
async function testPrompt() {
  loading.value = true;
  result.value = null;
  
  try {
    const response = await axios.post('/api/v1/prompt-test', form.value);
    result.value = response.data;
  } catch (error) {
    alert('测试失败：' + error.response?.data?.message || error.message);
  } finally {
    loading.value = false;
  }
}
</script>

<style scoped>
.prompt-test {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

.form-group select,
.form-group input,
.form-group textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.variable-input {
  margin-bottom: 15px;
  padding: 10px;
  background: #f5f5f5;
  border-radius: 4px;
}

button {
  padding: 10px 30px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.result {
  margin-top: 30px;
  padding: 20px;
  background: #f9f9f9;
  border-radius: 8px;
}

.result-section {
  margin-bottom: 20px;
}

.result-section h4 {
  margin-bottom: 10px;
  color: #333;
}

.result-section pre {
  background: white;
  padding: 15px;
  border-radius: 4px;
  overflow-x: auto;
  white-space: pre-wrap;
  word-wrap: break-word;
}

.result-section ul {
  list-style: none;
  padding: 0;
}

.result-section li {
  padding: 5px 0;
  border-bottom: 1px solid #eee;
}
</style>
```

### React 示例

```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function PromptTest() {
  const [form, setForm] = useState({
    templateName: 'content-optimization',
    model: 'gpt-3.5-turbo',
    variables: {},
    temperature: 0.7,
    maxTokens: 2000
  });
  
  const [models, setModels] = useState([]);
  const [result, setResult] = useState(null);
  const [loading, setLoading] = useState(false);
  
  useEffect(() => {
    loadModels();
    loadTemplateVariables();
  }, []);
  
  const loadModels = async () => {
    const response = await axios.get('/api/v1/prompt-test/models');
    setModels(response.data);
  };
  
  const loadTemplateVariables = async () => {
    const response = await axios.get(
      `/api/v1/prompt-test/template-variables/${form.templateName}`
    );
    setForm(prev => ({ ...prev, variables: response.data }));
  };
  
  const testPrompt = async () => {
    setLoading(true);
    setResult(null);
    
    try {
      const response = await axios.post('/api/v1/prompt-test', form);
      setResult(response.data);
    } catch (error) {
      alert('测试失败：' + (error.response?.data?.message || error.message));
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <div className="prompt-test">
      <h2>Prompt 测试工具</h2>
      
      {/* 表单内容类似 Vue 示例 */}
      
      <button onClick={testPrompt} disabled={loading}>
        {loading ? '测试中...' : '开始测试'}
      </button>
      
      {result && (
        <div className="result">
          <h3>测试结果</h3>
          <pre>{result.output}</pre>
        </div>
      )}
    </div>
  );
}

export default PromptTest;
```

## 使用场景

### 1. 模板调试

在开发新的 Prompt 模板时，可以快速验证不同变量组合的效果。

### 2. 模型对比

使用相同的 Prompt 测试不同模型，对比输出质量和性能。

### 3. 参数优化

调整 temperature 和 maxTokens 参数，找到最佳配置。

### 4. 变量测试

测试不同变量值对输出结果的影响。

## 最佳实践

1. **先测试后上线**：新模板在生产环境使用前，先在测试工具中验证
2. **保存测试用例**：记录好的测试用例，用于回归测试
3. **对比多个模型**：同一个 Prompt 在不同模型上测试，选择最合适的
4. **监控性能**：关注 Token 使用和执行时间，优化成本
5. **迭代优化**：根据测试结果不断优化 Prompt 模板

## 注意事项

- 测试会消耗实际的 API 调用额度
- 建议使用测试环境的 API Key
- 大量测试时注意 API 限流
- 敏感数据不要在测试中使用

