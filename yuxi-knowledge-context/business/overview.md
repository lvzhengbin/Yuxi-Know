# 业务概览

> 项目：yuxi-know v0.5.1 | 更新：2026-03-16

## 功能模块

### 模块 1: 智能体管理

- **职责**: 管理多种智能体的配置、运行和对话，支持子智能体、Skills、MCP 工具扩展
- **目录**: `src/agents/`
- **关键文件**:
  - `src/agents/chatbot/graph.py` — 基础对话智能体
  - `src/agents/deep_agent/` — 深度分析智能体（支持 todo/files 渲染）
  - `src/agents/reporter/` — 报告生成智能体
  - `src/agents/skills/` — Skills 扩展智能体
  - `src/agents/common/base.py` — BaseAgent 基类
  - `src/agents/common/middlewares/` — 中间件（知识库/运行时配置/Skills）

### 模块 2: 知识库（RAG）

- **职责**: 多格式文档上传、解析、分块、向量化入库，支持语义检索和 Rerank
- **目录**: `src/knowledge/`
- **关键文件**:
  - `src/knowledge/manager.py` — 知识库管理器（统一管理多类型实例）
  - `src/knowledge/factory.py` — 知识库工厂（按类型创建实例）
  - `src/knowledge/implementations/milvus.py` — Milvus 向量知识库
  - `src/knowledge/implementations/lightrag.py` — LightRAG 图谱知识库
  - `src/knowledge/implementations/dify.py` — Dify 知识库适配
  - `src/knowledge/chunking/` — 文档分块策略
  - `src/knowledge/indexing.py` — 文档索引入库

### 模块 3: 知识图谱

- **职责**: 基于 LightRAG 自动或手动构建知识图谱，支持可视化和智能体推理
- **目录**: `src/knowledge/graphs/`
- **关键文件**:
  - `server/routers/graph_router.py` — 图谱 API
  - `src/knowledge/implementations/lightrag.py` — LightRAG 实现
  - `web/src/views/GraphView.vue` — 图谱可视化（基于 AntV G6）

### 模块 4: 对话与会话管理

- **职责**: 管理用户对话历史、消息流式输出、会话持久化
- **目录**: `src/services/`
- **关键文件**:
  - `src/services/chat_stream_service.py` — 流式对话服务
  - `src/services/conversation_service.py` — 会话管理
  - `src/services/history_query_service.py` — 历史消息查询
  - `src/repositories/conversation_repository.py` — 会话数据访问

### 模块 5: MCP 扩展

- **职责**: 管理 Model Context Protocol 服务器配置，动态加载外部工具
- **目录**: `src/services/`
- **关键文件**:
  - `src/services/mcp_service.py` — MCP 工具加载与管理
  - `src/repositories/mcp_server_repository.py` — MCP 服务器配置存储
  - `server/routers/mcp_router.py` — MCP 管理 API

### 模块 6: Skills 扩展

- **职责**: 管理智能体 Skills（提示词注入、依赖展开、动态激活）
- **目录**: `src/services/`
- **关键文件**:
  - `src/services/skill_service.py` — Skill 服务
  - `src/agents/common/middlewares/skills_middleware.py` — Skills 中间件
  - `src/repositories/skill_repository.py` — Skill 数据访问

### 模块 7: 用户与权限管理

- **职责**: 用户注册/登录、部门管理、角色权限控制（superadmin/admin/user）
- **目录**: `src/repositories/` + `server/routers/`
- **关键文件**:
  - `src/storage/postgres/models_business.py` — User/Department ORM 模型
  - `src/repositories/user_repository.py` — 用户数据访问
  - `src/repositories/department_repository.py` — 部门数据访问
  - `server/routers/auth_router.py` — 认证 API
  - `server/utils/auth_middleware.py` — 鉴权中间件

### 模块 8: 任务队列

- **职责**: 异步处理耗时任务（文档解析、知识库入库等），支持任务状态追踪
- **目录**: `src/services/`
- **关键文件**:
  - `src/services/run_queue_service.py` — 任务队列服务
  - `src/services/run_worker.py` — 任务 Worker
  - `server/worker_main.py` — Worker 入口
  - `src/repositories/task_repository.py` — 任务数据访问

### 模块 9: 知识库评估

- **职责**: 支持导入评估基准或自动构建评估基准，对 RAG 检索质量进行评估
- **目录**: `src/services/`
- **关键文件**:
  - `src/services/evaluation_service.py` — 评估服务
  - `src/repositories/evaluation_repository.py` — 评估数据访问
  - `server/routers/evaluation_router.py` — 评估 API

### 模块 10: 系统配置与 Dashboard

- **职责**: 模型供应商配置、系统参数管理、运行状态统计
- **目录**: `src/config/` + `server/routers/`
- **关键文件**:
  - `src/config/app.py` — 应用配置（Pydantic + TOML）
  - `server/routers/system_router.py` — 系统配置 API
  - `server/routers/dashboard_router.py` — Dashboard 统计 API

---

## 关键业务流程

### 流程 1: 文档上传入库

- **入口**: `server/routers/knowledge_router.py` → POST `/api/knowledge/upload`
- **步骤**:
  1. 文件上传至 MinIO（`src/storage/minio/`）
  2. 创建异步任务（`src/services/task_service.py`）
  3. Worker 执行文档解析（PyMuPDF/docling/OCR）
  4. 文档分块（`src/knowledge/chunking/`）
  5. Embedding 向量化（`src/models/embed.py`）
  6. 写入 Milvus 或 LightRAG（`src/knowledge/indexing.py`）

### 流程 2: 智能体对话（流式）

- **入口**: `server/routers/chat_router.py` → POST `/api/chat/stream`
- **步骤**:
  1. 请求进入 `chat_stream_service.py`
  2. 加载智能体实例（`agent_run_service.py`）
  3. 执行 Middleware 链（知识库注入 → 工具配置 → 模型调用）
  4. LangGraph 流式输出 SSE 事件
  5. 对话持久化（`conversation_service.py`）

### 流程 3: 知识库检索

- **入口**: `src/agents/common/middlewares/knowledge_base_middleware.py`
- **步骤**:
  1. 智能体调用知识库工具
  2. `KnowledgeBaseManager` 路由到对应 KB 实例
  3. Milvus 向量检索 → 可选 Rerank（`src/models/rerank.py`）
  4. 返回相关文档片段给智能体

### 流程 4: 用户登录

- **入口**: `server/routers/auth_router.py` → POST `/api/auth/token`
- **步骤**:
  1. `LoginRateLimitMiddleware` 限流检查（10次/60秒）
  2. 验证用户名密码（`user_repository.py`）
  3. 检查账号锁定状态
  4. 生成 JWT Token 返回
