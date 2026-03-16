# 前端架构概览

> 领域：frontend-vue | 路径：web/ | 更新：2026-03-16

## 应用分层架构

```
┌─────────────────────────────────────────────────────┐
│                    路由层 (router/)                   │
│              Vue Router + 路由守卫                    │
│  权限控制：requiresAuth / requiresAdmin / requiresSuperAdmin │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   视图层 (views/)                     │
│  HomeView / AgentView / AgentSingleView              │
│  DataBaseView / DataBaseInfoView                     │
│  GraphView / DashboardView / ExtensionsView          │
│  LoginView / EmptyView                               │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  组件层 (components/)                 │
│                                                      │
│  对话类：AgentChatComponent / AgentInputArea          │
│          AgentMessageComponent / ChatSidebar         │
│  知识库：KnowledgeBaseCard / FileTable / FileUpload   │
│          ChunkParamsConfig / QuerySection            │
│  图谱：  GraphCanvas / GraphDetailPanel              │
│  扩展：  McpServersComponent / SkillsManager          │
│          ToolsManagerComponent                       │
│  系统：  UserManagement / DepartmentManagement       │
│          ModelProvidersComponent / SettingsModal     │
│  工具：  MarkdownContentViewer / TaskCenterDrawer    │
│          HumanApprovalModal / StatusBar              │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  状态层 (stores/)                     │
│                                                      │
│  agent.js      — 智能体列表、配置、对话状态            │
│  user.js       — 用户认证、角色权限                    │
│  database.js   — 知识库列表、文件状态                  │
│  graphStore.js — 图谱数据状态                         │
│  chatUI.js     — 对话 UI 状态（消息、滚动等）          │
│  config.js     — 系统配置状态                         │
│  info.js       — 应用信息/版本                        │
│  tasker.js     — 任务队列状态                         │
│  theme.js      — 主题（亮/暗模式）                     │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   API 层 (apis/)                     │
│                                                      │
│  base.js         — apiRequest 封装（认证/错误处理）   │
│  agent_api.js    — 智能体 + 聊天 API                 │
│  knowledge_api.js — 知识库 API                       │
│  graph_api.js    — 图谱 API                          │
│  dashboard_api.js — 统计数据 API                     │
│  system_api.js   — 系统配置 API                      │
│  mcp_api.js      — MCP 管理 API                      │
│  skill_api.js    — Skill 管理 API                    │
│  department_api.js — 部门管理 API                    │
│  tool_api.js     — 工具管理 API                      │
│  mindmap_api.js  — 思维导图 API                      │
└─────────────────────────────────────────────────────┘
```

## 目录结构职责

| 目录/文件 | 职责 |
|-----------|------|
| `web/src/views/` | 页面级组件，对应路由 |
| `web/src/components/` | 业务组件（可复用 UI 单元） |
| `web/src/stores/` | Pinia 状态管理（每模块一个 store） |
| `web/src/apis/` | 后端 API 接口封装 |
| `web/src/router/` | 路由定义 + 权限守卫 |
| `web/src/layouts/` | 布局组件（AppLayout / BlankLayout） |
| `web/src/composables/` | Vue Composition 逻辑复用 |
| `web/src/assets/css/` | 全局样式 + CSS 变量 |
| `web/src/utils/` | 工具函数 |

## 布局系统

两套布局：
- `AppLayout.vue`：带侧边导航的管理后台布局（管理员页面）
- `BlankLayout.vue`：无导航的纯内容布局（首页/登录）

代码位置：`web/src/layouts/`

## 权限体系

三级权限通过路由 meta 控制：

| meta 字段 | 含义 | 无权限时跳转 |
|-----------|------|-------------|
| `requiresAuth: true` | 需要登录 | `/login` |
| `requiresAdmin: true` | 需要 admin 或 superadmin | 默认智能体页面 |
| `requiresSuperAdmin: true` | 仅 superadmin | 默认智能体页面 |

代码位置：`web/src/router/index.js`

## 数据流：SSE 流式对话

```
用户输入
  → AgentInputArea.vue（组装消息）
  → agentStore.sendMessage()
  → agent_api.js sendAgentMessage()（fetch + ReadableStream）
  → useAgentStreamHandler.js（解析 SSE 事件）
  → chatUI store 更新消息状态
  → AgentMessageComponent.vue 实时渲染
```

## Composables

| 文件 | 职责 |
|------|------|
| `useAgentStreamHandler.js` | SSE 流解析与消息状态更新 |
| `useApproval.js` | Human-in-the-loop 审批交互 |
| `useGraph.js` | 知识图谱交互逻辑 |
| `useMention.js` | @提及功能（输入框知识库引用） |
