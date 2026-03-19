# 编码规范 — frontend-vue

## 命名约定

| 类型 | 规则 | 示例 |
|------|------|------|
| Vue 组件文件 | PascalCase | `AgentView.vue`, `ChatSidebarComponent.vue` |
| 组合式函数 | camelCase + use 前缀 | `useAgentStreamHandler`, `useGraph` |
| Pinia Store 文件 | camelCase | `agent.js`, `chatUI.js`, `graphStore.js` |
| API 文件 | snake_case + _api 后缀 | `agent_api.js`, `knowledge_api.js` |
| 普通函数 / 变量 | camelCase | `sendMessage()`, `agentList`, `currentModel` |
| CSS 类名 | kebab-case | `.chat-sidebar`, `.model-selector` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |

---

## 代码组织模式

### API 接口统一管理（强制要求）
```javascript
// ✅ 正确：在 src/apis/ 下定义接口，组件通过 import 调用
// src/apis/agent_api.js
export const getAgentList = () => request.get('/api/agents')

// 组件中
import { getAgentList } from '@/apis/agent_api'

// ❌ 错误：在组件内直接调用 axios
axios.get('/api/agents')
```

### Pinia Store 模式
```javascript
// ✅ 推荐：使用 defineStore + setup 风格
import { defineStore } from 'pinia'
export const useAgentStore = defineStore('agent', () => {
  const agentList = ref([])
  const currentAgent = ref(null)
  // ...
  return { agentList, currentAgent }
}, { persist: true })  // 持久化到 localStorage
```

### Composable 复用逻辑
```javascript
// ✅ 将跨组件共享的逻辑提取为 composable
// src/composables/useAgentStreamHandler.js
export function useAgentStreamHandler() {
  // SSE 流处理逻辑
  return { handleStream, isStreaming, ... }
}
```

---

## 样式规范

### 颜色变量（强制要求）
```less
// ✅ 正确：使用 base.css 中的颜色变量
.chat-bubble {
  background-color: var(--bg-color-primary);
  color: var(--text-color-primary);
}

// ❌ 错误：硬编码颜色值
.chat-bubble {
  background-color: #ffffff;
  color: #333333;
}
```

### UI 风格原则
- ❌ 禁止悬停位移动画（`transform: translateY(-2px)` 等）
- ❌ 避免过度使用阴影（`box-shadow` 仅在必要时使用）
- ❌ 避免渐变色（`background: linear-gradient(...)` 非必要不用）
- ✅ 保持简洁一致，与 Ant Design Vue 视觉风格统一

---

## 图标使用规范

```javascript
// ✅ 优先：@ant-design/icons-vue
import { SearchOutlined, SettingOutlined } from '@ant-design/icons-vue'

// ✅ 推荐（需注意尺寸）：lucide-vue-next
import { Search, Settings } from 'lucide-vue-next'
// 注意：lucide 图标默认尺寸可能与 Ant Design 不一致，需手动指定 size
```

---

## 错误处理

```javascript
// ✅ 统一错误处理：通过 errorHandler.js
import { handleError } from '@/utils/errorHandler'

try {
  await uploadFile(formData)
} catch (error) {
  handleError(error)  // 统一弹出 antd message 提示
}

// ✅ API 请求错误：在 base.js 拦截器中统一处理 401/500
```

---

## 反模式清单

### 反模式 1：API 散落在组件内
❌ 在 `.vue` 文件 `<script>` 中直接写 `axios.get('/api/...')`
✅ 所有接口定义在 `src/apis/` 目录下
**原因**: CLAUDE.md 明确规定，便于统一管理、修改 base URL、添加拦截器

### 反模式 2：硬编码颜色值
❌ `style="color: #666"` 或 `.text { color: #333 }`
✅ 使用 `base.css` 定义的 CSS 变量
**原因**: 破坏暗黑模式支持

### 反模式 3：Store 滥用
❌ 将仅在单个组件使用的状态放入 Pinia Store
✅ 局部状态用 `ref/reactive`，跨组件共享才放 Store
**原因**: Store 数据会被持久化（persist: true），产生不必要的 localStorage 占用

### 反模式 4：悬停位移动画
❌ 按钮/卡片悬停时添加位移动画
✅ 悬停仅改变背景色或透明度
**原因**: CLAUDE.md UI 规范明确禁止

---

## 开发规范

```bash
# 代码格式化（在 docker 容器内执行）
docker compose exec web npm run format

# 代码检查
docker compose exec web npm run lint
```

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
