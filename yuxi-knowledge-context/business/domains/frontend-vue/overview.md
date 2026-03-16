# 前端业务概览

> 领域：frontend-vue | 路径：web/ | 更新：2026-03-16

## 功能模块

### 模块 1: 智能体对话

- **职责**: 与智能体进行多轮对话，支持流式响应、文件上传、Human-in-the-loop 审批、工具调用结果展示
- **目录**: `web/src/views/AgentSingleView.vue`
- **关键文件**:
  - `web/src/components/AgentChatComponent.vue` — 对话主体
  - `web/src/components/AgentInputArea.vue` — 输入区域（含附件、@提及）
  - `web/src/components/AgentMessageComponent.vue` — 消息渲染（含 Markdown/Tool Call）
  - `web/src/components/HumanApprovalModal.vue` — 审批弹窗
  - `web/src/composables/useAgentStreamHandler.js` — SSE 流处理
  - `web/src/components/ToolCallingResult/` — 工具调用结果渲染

### 模块 2: 智能体管理（Admin）

- **职责**: 管理员查看和配置所有智能体，设置模型、系统提示词、工具、知识库、Skills
- **目录**: `web/src/views/AgentView.vue`
- **关键文件**:
  - `web/src/components/AgentPanel.vue` — 智能体列表面板
  - `web/src/components/AgentConfigSidebar.vue` — 配置侧边栏
  - `web/src/components/BasicSettingsSection.vue` — 基础设置
  - `web/src/components/KnowledgeSourceSection.vue` — 知识库配置
  - `web/src/components/ShareConfigForm.vue` — 分享配置
  - `web/src/stores/agent.js` — 智能体状态管理

### 模块 3: 知识库管理

- **职责**: 创建/管理知识库，上传文档，配置分块参数，查询检索，知识库评估
- **目录**: `web/src/views/DataBaseView.vue` / `DataBaseInfoView.vue`
- **关键文件**:
  - `web/src/components/KnowledgeBaseCard.vue` — 知识库卡片
  - `web/src/components/DatabaseHeader.vue` — 知识库详情头部
  - `web/src/components/FileTable.vue` — 文件列表
  - `web/src/components/FileUploadModal.vue` — 文件上传弹窗
  - `web/src/components/FileTreeComponent.vue` — 文件树
  - `web/src/components/ChunkParamsConfig.vue` — 分块参数配置
  - `web/src/components/QuerySection.vue` — 检索测试
  - `web/src/components/EvaluationBenchmarks.vue` — RAG 评估
  - `web/src/components/RAGEvaluationTab.vue` — 评估标签页
  - `web/src/apis/knowledge_api.js` — 知识库 API

### 模块 4: 知识图谱

- **职责**: 可视化展示知识图谱，支持节点检索、属性查看、图谱操作
- **目录**: `web/src/views/GraphView.vue`
- **关键文件**:
  - `web/src/components/GraphCanvas.vue` — AntV G6 图谱画布
  - `web/src/components/GraphDetailPanel.vue` — 节点/边详情面板
  - `web/src/components/KnowledgeGraphSection.vue` — 图谱配置区
  - `web/src/composables/useGraph.js` — 图谱交互逻辑
  - `web/src/stores/graphStore.js` — 图谱数据状态

### 模块 5: 扩展管理（Admin）

- **职责**: 管理 MCP 服务器、Skills 扩展、工具配置
- **目录**: `web/src/views/ExtensionsView.vue`
- **关键文件**:
  - `web/src/components/McpServersComponent.vue` — MCP 服务器管理
  - `web/src/components/McpEnvEditor.vue` — MCP 环境变量编辑
  - `web/src/components/SkillsManagerComponent.vue` — Skills 管理
  - `web/src/components/ToolsManagerComponent.vue` — 工具管理

### 模块 6: Dashboard 统计

- **职责**: 系统运行状态监控、对话统计、使用量分析
- **目录**: `web/src/views/DashboardView.vue`
- **关键文件**:
  - `web/src/components/dashboard/` — Dashboard 子组件
  - `web/src/components/ProjectOverview.vue` — 项目概览
  - `web/src/apis/dashboard_api.js` — 统计数据 API

### 模块 7: 用户与权限管理（SuperAdmin）

- **职责**: 用户账号管理、部门管理、模型供应商配置
- **目录**: 嵌入于 `ExtensionsView.vue` 或 `SettingsModal`
- **关键文件**:
  - `web/src/components/UserManagementComponent.vue` — 用户管理
  - `web/src/components/DepartmentManagementComponent.vue` — 部门管理
  - `web/src/components/ModelProvidersComponent.vue` — 模型供应商配置
  - `web/src/stores/user.js` — 用户认证状态

### 模块 8: 思维导图

- **职责**: 基于知识库文件生成思维导图并可视化展示
- **目录**: 组件嵌入于知识库详情页
- **关键文件**:
  - `web/src/components/MindMapSection.vue` — 思维导图展示
  - `web/src/apis/mindmap_api.js` — 思维导图 API

---

## 关键业务流程

### 流程 1: 用户登录与权限路由

- **入口**: `web/src/views/LoginView.vue`
- **步骤**:
  1. 用户输入 loginId + 密码
  2. `userStore.login()` 调用 `/api/auth/token`
  3. 获取 JWT Token + 用户角色，存入 localStorage
  4. 路由守卫（`router/index.js beforeEach`）根据角色跳转：
     - superadmin/admin → 管理页面（`/agent`、`/database` 等）
     - 普通用户 → 默认智能体页面（`/agent/:agent_id`）

### 流程 2: 流式对话

- **入口**: `AgentSingleView.vue` / `AgentChatComponent.vue`
- **步骤**:
  1. 用户在 `AgentInputArea` 输入消息（支持文件附件、@知识库）
  2. `agentStore` 调用 `agent_api.sendAgentMessage()` 发起 fetch SSE
  3. `useAgentStreamHandler` 解析 SSE 事件流
  4. 实时更新 `chatUI store` 中的消息列表
  5. `AgentMessageComponent` 渲染 Markdown + 工具调用结果
  6. 如遇 `HumanApproval` 事件，弹出 `HumanApprovalModal` 等待确认

### 流程 3: 文件上传入库

- **入口**: `web/src/components/FileUploadModal.vue`
- **步骤**:
  1. 用户选择文件（支持文件夹/压缩包）
  2. 调用 `knowledge_api.uploadFile()`
  3. 后端创建异步任务，返回任务 ID
  4. `TaskCenterDrawer` 轮询任务状态展示进度

### 流程 4: 知识图谱构建与可视化

- **入口**: `web/src/views/GraphView.vue`
- **步骤**:
  1. 选择 LightRAG 类型知识库
  2. 调用 `graph_api` 获取图谱数据
  3. `GraphCanvas` 使用 AntV G6 渲染节点/边
  4. `useGraph` composable 处理交互（搜索、选中、详情展示）
