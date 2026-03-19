# 架构总览 — frontend-vue

## 架构模式

本项目采用**MVVM + 模块化 SPA** 模式，基于 Vue 3 Composition API，Views 负责展示，Stores 管理全局状态，Composables 封装可复用逻辑，APIs 统一管理接口调用。

---

## 架构图

```
┌─────────────────────────────────────────────────────┐
│                    Views（页面层）                    │
│  AgentView · DataBaseView · GraphView                │
│  DashboardView · ExtensionsView · LoginView          │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
┌──────────▼──────────┐  ┌────────────▼──────────────┐
│   Components（组件）  │  │   Composables（逻辑复用）  │
│  ChatSidebarComponent│  │  useAgentStreamHandler    │
│  ModelSelectorComp.  │  │  useGraph                 │
│  McpServersComponent │  │  useApproval              │
│  ToolCallingResult/  │  │  useMention               │
│  MindMapSection      │  └────────────┬──────────────┘
└──────────┬───────────┘               │
           │              ┌────────────▼──────────────┐
           │              │   Stores（Pinia 状态管理）  │
           │              │  agent · database · user   │
           │              │  graphStore · chatUI       │
           │              │  config · tasker · theme   │
           └──────────────┴────────────┬──────────────┘
                                       │
                          ┌────────────▼──────────────┐
                          │   APIs（接口层）             │
                          │  agent_api · knowledge_api  │
                          │  graph_api · tool_api       │
                          │  mcp_api · system_api       │
                          │  dashboard_api · skill_api  │
                          └────────────┬──────────────┘
                                       │
                          ┌────────────▼──────────────┐
                          │   后端 FastAPI              │
                          │   /api/* (SSE + REST)       │
                          └───────────────────────────┘
```

---

## 项目目录结构

```
web/src/
├── main.js              # 应用入口，注册 Vue/Router/Pinia/Antd
├── App.vue              # 根组件，路由出口
├── router/              # Vue Router 路由定义
│
├── views/               # 页面级组件（路由直接映射）
│   ├── AgentView.vue    # 智能体对话主界面（核心页面）
│   ├── AgentSingleView.vue  # 单智能体嵌入式视图
│   ├── DataBaseView.vue     # 知识库列表管理
│   ├── DataBaseInfoView.vue # 知识库详情（文件管理、配置）
│   ├── GraphView.vue        # 知识图谱可视化
│   ├── DashboardView.vue    # 仪表盘（系统状态）
│   ├── ExtensionsView.vue   # 扩展管理（工具/MCP/Skills）
│   ├── HomeView.vue         # 首页
│   └── LoginView.vue        # 登录页
│
├── components/          # 可复用组件
│   ├── ChatSidebarComponent.vue    # 对话侧边栏（历史会话）
│   ├── ModelSelectorComponent.vue  # 模型选择器
│   ├── McpServersComponent.vue     # MCP 服务配置
│   ├── MarkdownContentViewer.vue   # Markdown 渲染
│   ├── MindMapSection.vue          # 思维导图
│   ├── RefsComponent.vue           # 引用来源展示
│   └── ToolCallingResult/          # 工具调用结果渲染
│       └── tools/                  # 各类工具渲染组件
│           ├── MysqlListTablesTool.vue
│           ├── TodoListTool.vue
│           ├── WriteFileTool.vue
│           └── AskUserQuestionTool.vue
│
├── stores/              # Pinia 状态管理（持久化配置）
│   ├── agent.js         # 智能体列表、当前 Agent 配置
│   ├── database.js      # 知识库列表
│   ├── user.js          # 用户信息、登录态
│   ├── graphStore.js    # 图谱数据
│   ├── chatUI.js        # 对话 UI 状态（消息列表、加载状态）
│   ├── config.js        # 应用全局配置
│   ├── tasker.js        # 异步任务状态
│   ├── theme.js         # 主题（暗黑/亮色）
│   └── info.js          # 系统信息（版本等）
│
├── apis/                # API 接口定义（统一管理）
│   ├── base.js          # axios 实例、拦截器（Token 注入）
│   ├── agent_api.js     # 智能体相关接口
│   ├── knowledge_api.js # 知识库接口（上传/查询/评估）
│   ├── graph_api.js     # 知识图谱接口
│   ├── tool_api.js      # 工具管理接口
│   ├── mcp_api.js       # MCP 服务接口
│   ├── skill_api.js     # Skills 接口
│   ├── system_api.js    # 系统配置接口
│   ├── dashboard_api.js # 仪表盘接口
│   └── tasker.js        # 任务查询接口
│
├── composables/         # 组合式逻辑（可复用）
│   ├── useAgentStreamHandler.js  # SSE 流式响应处理（核心）
│   ├── useGraph.js               # 图谱交互逻辑
│   ├── useApproval.js            # human-in-the-loop 审批
│   └── useMention.js             # @ 提及功能
│
└── utils/               # 工具函数
    ├── messageProcessor.js   # 消息格式转换
    ├── kb_utils.js           # 知识库工具函数
    ├── agentValidator.js     # Agent 配置校验
    ├── chatExporter.js       # 对话导出
    ├── errorHandler.js       # 全局错误处理
    ├── modelIcon.js          # 模型 Logo 映射
    └── chunk_presets.js      # 分块预设配置
```

---

## 核心模块

### AgentView.vue
- **职责**: 智能体对话核心页面，整合消息列表、输入框、工具调用展示、模型选择
- **依赖**: `useAgentStreamHandler`、`chatUI store`、`agent store`
- **代码位置**: `web/src/views/AgentView.vue`

### useAgentStreamHandler.js
- **职责**: 处理后端 SSE 流式响应，解析 LangGraph 事件流，更新消息状态
- **关键**: 支持工具调用结果、`human-in-the-loop` 中断、消息追加
- **代码位置**: `web/src/composables/useAgentStreamHandler.js`

### apis/base.js
- **职责**: axios 实例封装，统一注入 JWT Token，处理 401 跳转登录
- **代码位置**: `web/src/apis/base.js`

---

## 数据流

### 对话消息流
```
用户输入 (AgentView)
  → agent_api.sendMessage() → POST /api/chat/stream (SSE)
    → useAgentStreamHandler 监听 SSE 事件
      → 解析 event: message / tool_call / interrupt
      → chatUI store.addMessage() / appendChunk()
        → 触发 Vue 响应式更新
          → MarkdownContentViewer 实时渲染
```

### 知识库上传流
```
文件拖拽 (DataBaseInfoView)
  → knowledge_api.uploadFile()
  → tasker store 轮询任务状态
    → 任务完成 → 刷新文件列表
```

---

## 关键设计决策

- **所有 API 接口统一在 `src/apis/` 定义**（CLAUDE.md 规定），禁止在组件内直接调用 axios。
- **图标策略**: 优先使用 `@ant-design/icons-vue`，补充使用 `lucide-vue-next`（需注意尺寸适配）。
- **样式策略**: 使用 Less + `base.css` CSS 变量，禁止硬编码颜色，支持暗黑模式。
- **UI 原则**: 简洁一致，不做悬停位移动画，不过度使用阴影和渐变色。

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
