# backend-python 项目工程概览

## 工程模块结构

| 模块名 | 目录路径 | 职责简述 |
|--------|---------|---------|
| HTTP 入口层 | `server/` | FastAPI 应用、路由注册、中间件（鉴权/限流/日志） |
| Agent 领域 | `src/agents/` | 智能体定义、LangGraph 编排、中间件链、工具集 |
| Knowledge 领域 | `src/knowledge/` | 知识库管理、文档分块、向量检索、图谱 RAG |
| Service 编排层 | `src/services/` | 对话流、任务调度、MCP 管理、评估服务 |
| Repository 层 | `src/repositories/` | 数据访问对象（知识库、会话、Agent 配置等） |
| Storage 层 | `src/storage/` | 数据库 ORM 模型定义（PostgreSQL） |
| Config 层 | `src/config/` | 应用配置（Pydantic + TOML 持久化）、模型列表 |
| Plugins 层 | `src/plugins/` | 文档解析插件工厂（OCR/docling/MinerU） |
| Utils 层 | `src/utils/` | 日志、Prompt 模板、工具函数 |
| Worker 入口 | `server/worker_main.py` | ARQ 异步任务 Worker（文档入库等） |

---

## 项目目录树（核心）

```
yuxi-know/
├── server/                         # HTTP 入口层
│   ├── main.py                     # FastAPI app, 中间件注册（鉴权/限流/CORS/日志）
│   ├── worker_main.py              # ARQ Worker 入口
│   ├── routers/
│   │   ├── chat_router.py          # /api/chat/* 对话接口（SSE 流式）
│   │   ├── knowledge_router.py     # /api/knowledge/* 知识库 CRUD
│   │   ├── auth_router.py          # /api/auth/* 登录/Token
│   │   ├── graph_router.py         # /api/graph/* 知识图谱
│   │   ├── tool_router.py          # /api/tools/* 工具管理
│   │   ├── mcp_router.py           # /api/mcp/* MCP 服务
│   │   ├── task_router.py          # /api/tasks/* 异步任务
│   │   ├── skill_router.py         # /api/skills/* Skills
│   │   ├── evaluation_router.py    # /api/evaluation/* RAG 评估
│   │   ├── dashboard_router.py     # /api/dashboard/* 系统状态
│   │   ├── mindmap_router.py       # /api/mindmap/* 思维导图
│   │   └── department_router.py    # /api/department/* 部门管理
│   └── utils/
│       ├── auth_middleware.py      # 鉴权中间件、公开路径白名单
│       ├── access_log_middleware.py# 请求耗时日志
│       ├── lifespan.py             # 应用启动/关闭钩子（DB 初始化等）
│       ├── auth_utils.py           # JWT 工具
│       └── user_utils.py           # 用户工具
│
└── src/
    ├── agents/
    │   ├── common/
    │   │   ├── base.py             # BaseAgent 抽象类（LangGraph 编排基础）
    │   │   ├── state.py            # LangGraph State 定义
    │   │   ├── context.py          # BaseContext（运行时上下文）
    │   │   ├── models.py           # Agent 数据模型
    │   │   ├── utils.py            # Agent 工具函数
    │   │   ├── middlewares/        # Agent 中间件（运行前上下文注入）
    │   │   │   ├── knowledge_base_middleware.py
    │   │   │   ├── dynamic_tool_middleware.py
    │   │   │   ├── attachment_middleware.py
    │   │   │   ├── summary_middleware.py
    │   │   │   └── runtime_config_middleware.py
    │   │   ├── toolkits/           # 工具集实现
    │   │   │   ├── registry.py     # 工具注册中心
    │   │   │   ├── kbs/tools.py    # 知识库检索工具
    │   │   │   ├── mysql/          # MySQL 查询工具
    │   │   │   ├── buildin/tools.py# 内置工具
    │   │   │   └── debug/tools.py  # 调试工具
    │   │   └── backends/           # 后端适配（skills_backend, composite）
    │   ├── chatbot/graph.py        # ChatbotAgent 实现
    │   ├── reporter/graph.py       # ReporterAgent 实现
    │   └── deep_agent/             # DeepAgent（深度分析）
    │
    ├── knowledge/
    │   ├── base.py                 # KnowledgeBase 抽象基类
    │   ├── manager.py              # KnowledgeBaseManager（统一管理入口）
    │   ├── factory.py              # 知识库工厂方法
    │   ├── indexing.py             # 文档索引入库流程
    │   ├── implementations/
    │   │   ├── milvus.py           # Milvus 向量检索实现
    │   │   ├── lightrag.py         # LightRAG 图谱 RAG 实现
    │   │   └── dify.py             # Dify 外部知识库适配
    │   ├── chunking/ragflow_like/  # RAGFlow 风格文档分块器
    │   │   ├── dispatcher.py       # 分块器分发
    │   │   ├── parsers/            # 场景化解析（laws/book/qa/general）
    │   │   └── presets.py          # 分块预设配置
    │   └── graphs/
    │       ├── upload_graph_service.py  # 图谱上传服务
    │       └── adapters/               # 图谱适配器（LightRAG）
    │
    ├── services/
    │   ├── chat_stream_service.py  # 对话流编排（SSE 核心）
    │   ├── agent_run_service.py    # Agent 运行调度
    │   ├── conversation_service.py # 会话管理
    │   ├── task_service.py         # 异步任务
    │   ├── mcp_service.py          # MCP 连接管理
    │   ├── tool_service.py         # 工具服务
    │   ├── evaluation_service.py   # RAG 评估
    │   ├── run_queue_service.py    # 任务队列服务
    │   └── doc_converter.py        # 文档格式转换
    │
    ├── repositories/               # 数据访问对象（Repository 模式）
    ├── storage/postgres/           # SQLAlchemy ORM 模型
    ├── config/                     # 全局配置
    ├── plugins/                    # 文档解析插件
    └── utils/                      # 工具函数
```

---

## 模块依赖关系

```
server/routers  ─→  src/services  ─→  src/agents
                                  ─→  src/knowledge
                                  ─→  src/repositories
src/agents      ─→  src/agents/common/middlewares  ─→  src/knowledge
src/knowledge   ─→  src/repositories  ─→  src/storage
src/config      ←─  (全局单例，被所有层读取)
src/plugins     ←─  src/knowledge/indexing (文档解析时调用)
```

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
