# 架构总览 — backend-python

## 架构模式

本项目采用**分层架构 + 领域驱动**的混合模式，按职责清晰分层：HTTP 入口层 → 服务编排层 → 领域核心层 → 基础设施层。

---

## 架构图

```
┌──────────────────────────────────────────────────────────┐
│                    HTTP / SSE 入口层                       │
│  server/routers/  (FastAPI Router)                        │
│  chat_router · knowledge_router · auth_router             │
│  graph_router · tool_router · mcp_router · task_router    │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│                    服务编排层                              │
│  src/services/                                            │
│  chat_stream_service · agent_run_service                  │
│  conversation_service · knowledge_service                 │
│  task_service · mcp_service · tool_service                │
└──────────────┬───────────────────────┬───────────────────┘
               │                       │
┌──────────────▼──────┐   ┌────────────▼──────────────────┐
│   Agent 领域层       │   │   Knowledge 领域层             │
│   src/agents/        │   │   src/knowledge/               │
│   BaseAgent          │   │   KnowledgeBaseManager         │
│   ChatbotAgent       │   │   implementations/             │
│   ReporterAgent      │   │     milvus · lightrag · dify   │
│   DeepAgent          │   │   chunking/ragflow_like/        │
│   common/            │   │   graphs/ (LightRAG 图谱)      │
│     middlewares/     │   │   indexing · factory           │
│     toolkits/        │   └────────────────────────────────┘
│     context/state    │
└──────────────┬───────┘
               │
┌──────────────▼───────────────────────────────────────────┐
│                   基础设施层                               │
│  src/repositories/    数据访问对象（Repository 模式）      │
│  src/storage/         数据库驱动（postgres / sqlite）      │
│  src/config/          应用配置（TOML 持久化）              │
│  src/plugins/         文档解析插件（OCR / docling）        │
│  src/utils/           通用工具（logger / prompts / image） │
└──────────────────────────────────────────────────────────┘

外部依赖：
  PostgreSQL · SQLite · Milvus · Neo4j · Redis · MinIO
  OpenAI/SiliconFlow · DeepSeek · DashScope · MCP Servers
```

---

## 项目目录结构

```
yuxi-know/
├── server/                  # HTTP 入口层
│   ├── main.py              # FastAPI 应用实例、中间件注册
│   ├── routers/             # API 路由（每个业务域一个文件）
│   │   ├── chat_router.py   # 对话流 / SSE 流式输出
│   │   ├── knowledge_router.py  # 知识库 CRUD
│   │   ├── auth_router.py   # 登录认证
│   │   ├── graph_router.py  # 知识图谱
│   │   ├── tool_router.py   # 工具管理
│   │   ├── mcp_router.py    # MCP 服务配置
│   │   ├── task_router.py   # 异步任务
│   │   ├── skill_router.py  # Skills 管理
│   │   ├── evaluation_router.py # RAG 评估
│   │   └── dashboard_router.py  # 仪表盘
│   ├── utils/               # 服务层工具
│   │   ├── auth_middleware.py   # 鉴权中间件
│   │   ├── access_log_middleware.py  # 访问日志
│   │   ├── lifespan.py      # 应用启动/关闭钩子
│   │   └── auth_utils.py    # JWT 工具函数
│   └── worker_main.py       # ARQ Worker 入口
│
├── src/                     # 领域核心层 + 基础设施层
│   ├── agents/              # 智能体领域
│   │   ├── common/          # 共享基础设施
│   │   │   ├── base.py      # BaseAgent 抽象类
│   │   │   ├── state.py     # LangGraph 状态定义
│   │   │   ├── context.py   # BaseContext（运行时上下文）
│   │   │   ├── middlewares/ # Agent 中间件
│   │   │   │   ├── knowledge_base_middleware.py  # 知识库注入
│   │   │   │   ├── dynamic_tool_middleware.py    # 动态工具
│   │   │   │   ├── attachment_middleware.py      # 附件处理
│   │   │   │   ├── summary_middleware.py         # 摘要生成
│   │   │   │   └── runtime_config_middleware.py  # 运行时配置
│   │   │   └── toolkits/    # 工具集注册与实现
│   │   │       ├── registry.py   # 工具注册中心
│   │   │       ├── kbs/          # 知识库工具
│   │   │       ├── mysql/        # MySQL 查询工具
│   │   │       └── buildin/      # 内置工具（网络搜索等）
│   │   ├── chatbot/         # 通用对话 Agent
│   │   ├── reporter/        # 报告生成 Agent
│   │   └── deep_agent/      # 深度分析 Agent（DeepAgents）
│   │
│   ├── knowledge/           # 知识库领域
│   │   ├── base.py          # KnowledgeBase 抽象基类
│   │   ├── manager.py       # KnowledgeBaseManager（统一管理）
│   │   ├── factory.py       # 工厂方法（按 kb_type 实例化）
│   │   ├── indexing.py      # 文档索引入库流程
│   │   ├── implementations/ # 具体知识库实现
│   │   │   ├── milvus.py    # Milvus 向量检索
│   │   │   ├── lightrag.py  # LightRAG 图谱 RAG
│   │   │   └── dify.py      # Dify 外部知识库
│   │   ├── chunking/        # 文档分块
│   │   │   └── ragflow_like/ # RAGFlow 风格分块器
│   │   └── graphs/          # 知识图谱上传与管理
│   │       └── adapters/    # 图谱适配器（LightRAG 等）
│   │
│   ├── services/            # 服务编排层
│   │   ├── chat_stream_service.py   # 对话流编排（SSE 输出核心）
│   │   ├── agent_run_service.py     # Agent 运行调度
│   │   ├── conversation_service.py  # 会话管理
│   │   ├── task_service.py          # 异步任务
│   │   ├── mcp_service.py           # MCP 连接管理
│   │   ├── tool_service.py          # 工具服务
│   │   ├── evaluation_service.py    # RAG 评估
│   │   └── run_queue_service.py     # 任务队列
│   │
│   ├── repositories/        # 数据访问层（Repository 模式）
│   │   ├── knowledge_base_repository.py
│   │   ├── conversation_repository.py
│   │   ├── agent_config_repository.py
│   │   └── mcp_server_repository.py
│   │
│   ├── storage/             # 数据库驱动
│   │   └── postgres/        # PostgreSQL 模型定义
│   │
│   ├── config/              # 全局配置
│   │   ├── app.py           # Config（Pydantic，TOML 持久化）
│   │   └── static/models.py # 预设模型列表
│   │
│   ├── plugins/             # 文档解析插件
│   │   ├── document_processor_factory.py  # 工厂分发
│   │   ├── rapid_ocr_processor.py         # RapidOCR
│   │   ├── deepseek_ocr_parser.py         # DeepSeek OCR
│   │   └── mineru_official_parser.py      # MinerU 解析
│   │
│   └── utils/               # 通用工具
│       ├── logging_config.py  # Loguru 日志配置
│       ├── prompts.py         # Prompt 模板
│       └── image_processor.py # 图像处理
```

---

## 核心模块

### server/main.py
- **职责**: FastAPI 应用入口，注册路由（`/api` 前缀）、CORS、鉴权中间件、限流中间件
- **关键**: `LoginRateLimitMiddleware`（登录限频 10次/分钟）、`AuthMiddleware`（JWT 校验）
- **代码位置**: `server/main.py`

### src/agents/common/base.py — BaseAgent
- **职责**: 所有 Agent 的抽象基类，封装 LangGraph 编排、checkpointer 管理、流式输出
- **关键方法**: `stream_messages()`（SSE 输出）、`get_graph()`（抽象，子类实现）、`_get_checkpointer()`（SQLite/PostgreSQL 双模式）
- **代码位置**: `src/agents/common/base.py`

### src/knowledge/manager.py — KnowledgeBaseManager
- **职责**: 统一管理多种类型知识库实例，直接通过 Repository 访问数据库
- **关键**: 支持 milvus / lightrag / dify 三种知识库类型，通过 `factory.py` 工厂实例化
- **代码位置**: `src/knowledge/manager.py`

### src/services/chat_stream_service.py
- **职责**: 对话流编排核心，处理 Agent 流式输出、附件注入、会话持久化
- **关键**: SSE 实时推送 `AIMessageChunk`，支持 `human-in-the-loop` 中断恢复
- **代码位置**: `src/services/chat_stream_service.py`

### src/config/app.py — Config
- **职责**: 全局配置单例，Pydantic BaseModel，从 `saves/config/base.toml` 加载用户配置
- **关键**: 模型提供商状态检测（环境变量），支持自定义 Provider（`custom_providers.toml`）
- **代码位置**: `src/config/app.py`

---

## 模块依赖关系

```
server/routers → src/services → src/agents → src/repositories
                              → src/knowledge → src/repositories
                              → src/storage
server/routers → src/config (全局单例)
src/agents/common/middlewares → src/knowledge (运行时注入知识库)
src/knowledge/manager → src/repositories → src/storage
src/plugins → (外部: docling, pymupdf, rapidocr)
```

---

## 数据流

### 对话请求数据流
```
POST /api/chat/stream (chat_router)
  → chat_stream_service.stream()
    → AgentConfigRepository 读取 agent 配置
    → agent_manager.get_agent(agent_id)
    → BaseAgent.stream_messages()
      → 中间件链处理 (knowledge_base / tools / attachment)
      → LangGraph graph.astream()
        → 各 Node 调用 LLM / Tools / KnowledgeBase
    → SSE 推送 AIMessageChunk
  → ConversationRepository 保存会话
```

### 文档入库数据流
```
POST /api/knowledge/upload (knowledge_router)
  → ARQ 异步任务队列 (task_service)
    → 文档解析 (plugins/document_processor_factory)
      → docling / rapidocr / mineru
    → 文档分块 (knowledge/chunking/ragflow_like)
    → 向量化 + 入库 (knowledge/implementations/milvus 或 lightrag)
      → Milvus 存储向量
      → Neo4j / LightRAG 构建知识图谱
```

---

## 关键设计决策

- **为什么采用分层架构？** Router 层只做 HTTP 协议处理，业务逻辑全部在 Service 层，便于单元测试和复用。
- **为什么 Repository 模式？** 隔离数据访问逻辑，Service 层不直接操作 ORM，便于切换存储后端。
- **中间件链设计**: Agent 运行前通过中间件链动态注入知识库、工具、附件等上下文，保持 Agent 核心图的纯净。

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
