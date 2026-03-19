# frontend-vue 项目工程概览

## 工程模块结构

| 模块名 | 目录路径 | 职责简述 |
|--------|---------|---------|
| 入口 | `web/src/main.js` | 应用初始化，注册 Vue/Pinia/Router/Antd |
| 页面层 | `web/src/views/` | 路由级页面组件 |
| 组件层 | `web/src/components/` | 可复用 UI 组件 |
| 状态管理 | `web/src/stores/` | Pinia Store（持久化） |
| API 层 | `web/src/apis/` | 所有后端接口定义（强制统一管理） |
| 组合逻辑 | `web/src/composables/` | 跨组件复用的组合式函数 |
| 工具函数 | `web/src/utils/` | 消息处理、导出、校验等 |
| 静态资源 | `web/src/assets/` | CSS 变量、全局样式 |

---

## 项目目录树（核心）

```
web/src/
├── main.js                          # 应用入口
├── App.vue                          # 根组件
│
├── views/                           # 页面（路由级）
│   ├── AgentView.vue                # 智能体对话主界面
│   ├── AgentSingleView.vue          # 单智能体嵌入式视图
│   ├── DataBaseView.vue             # 知识库列表
│   ├── DataBaseInfoView.vue         # 知识库详情（文件/配置）
│   ├── GraphView.vue                # 知识图谱可视化
│   ├── DashboardView.vue            # 仪表盘
│   ├── ExtensionsView.vue           # 扩展管理（工具/MCP/Skills）
│   ├── HomeView.vue                 # 首页
│   ├── LoginView.vue                # 登录
│   └── EmptyView.vue                # 空状态占位
│
├── components/
│   ├── ChatSidebarComponent.vue     # 对话历史侧边栏
│   ├── ModelSelectorComponent.vue   # 模型选择器
│   ├── McpServersComponent.vue      # MCP 服务管理
│   ├── McpEnvEditor.vue             # MCP 环境变量编辑
│   ├── MarkdownContentViewer.vue    # Markdown 渲染组件
│   ├── MindMapSection.vue           # 思维导图展示
│   ├── RefsComponent.vue            # 检索来源引用展示
│   ├── WebSearchSourceSection.vue   # 网络搜索来源展示
│   └── ToolCallingResult/           # 工具调用结果渲染
│       └── tools/                   # 各工具专用渲染组件
│           ├── MysqlListTablesTool.vue
│           ├── MysqlDescribeTableTool.vue
│           ├── TodoListTool.vue
│           ├── WriteFileTool.vue
│           ├── EditFileTool.vue
│           ├── SearchFileContentTool.vue
│           ├── ImageTool.vue
│           ├── TaskTool.vue
│           ├── CalculatorTool.vue
│           └── AskUserQuestionTool.vue
│
├── stores/                          # Pinia 状态管理
│   ├── agent.js                     # 智能体列表与配置
│   ├── database.js                  # 知识库列表
│   ├── user.js                      # 用户信息、登录态
│   ├── graphStore.js                # 图谱数据
│   ├── chatUI.js                    # 对话 UI 状态
│   ├── config.js                    # 系统配置
│   ├── tasker.js                    # 异步任务状态
│   ├── theme.js                     # 主题（暗黑/亮色）
│   └── info.js                      # 系统信息
│
├── apis/                            # 接口层（禁止在组件中直接调用 axios）
│   ├── base.js                      # axios 实例、Token 注入拦截器
│   ├── index.js                     # 统一导出
│   ├── agent_api.js                 # 智能体 CRUD、对话、配置
│   ├── knowledge_api.js             # 知识库 CRUD、文件上传、评估
│   ├── graph_api.js                 # 知识图谱 CRUD、可视化数据
│   ├── tool_api.js                  # 工具管理
│   ├── mcp_api.js                   # MCP 服务配置
│   ├── skill_api.js                 # Skills 管理
│   ├── system_api.js                # 系统配置
│   ├── dashboard_api.js             # 仪表盘数据
│   ├── mindmap_api.js               # 思维导图
│   ├── department_api.js            # 部门管理
│   └── tasker.js                    # 任务查询
│
├── composables/                     # 组合式逻辑
│   ├── useAgentStreamHandler.js     # SSE 流式响应解析（核心）
│   ├── useGraph.js                  # 图谱交互（G6 操作）
│   ├── useApproval.js               # human-in-the-loop 审批交互
│   └── useMention.js                # @ 提及功能
│
└── utils/
    ├── messageProcessor.js          # 消息格式转换（LangGraph → UI）
    ├── kb_utils.js                  # 知识库工具函数
    ├── agentValidator.js            # Agent 配置校验
    ├── chatExporter.js              # 对话导出（HTML 格式）
    ├── errorHandler.js              # 全局错误处理
    ├── modelIcon.js                 # 模型 Logo / 图标映射
    └── chunk_presets.js             # 知识库分块预设
```

---

## 模块依赖关系

```
Views  ─→  Composables  ─→  APIs  ─→  后端
Views  ─→  Stores       ─→  APIs
Views  ─→  Components   ─→  Stores
```

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
