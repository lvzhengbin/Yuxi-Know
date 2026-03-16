# 架构概览

> 项目：yuxi-know v0.5.1 | 更新：2026-03-16

## 系统分层架构

```
┌─────────────────────────────────────────────────────┐
│                    前端层 (web/)                      │
│         Vue.js + Vite + Ant Design Vue               │
│   views/ │ components/ │ stores/ │ apis/             │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP / SSE
┌──────────────────────▼──────────────────────────────┐
│                   API 网关层 (server/)                │
│              FastAPI + Middleware                     │
│  AuthMiddleware │ RateLimitMiddleware │ CORS          │
│  routers/: chat │ knowledge │ graph │ agent │ ...    │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  服务层 (src/services/)               │
│  chat_stream_service │ agent_run_service             │
│  conversation_service │ knowledge_service            │
│  task_service │ mcp_service │ skill_service          │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  智能体层 (src/agents/)               │
│                                                      │
│  ┌─────────────┐  ┌────────────┐  ┌──────────────┐  │
│  │ ChatbotAgent│  │ DeepAgent  │  │ ReporterAgent│  │
│  └──────┬──────┘  └─────┬──────┘  └──────┬───────┘  │
│         └───────────────┼────────────────┘          │
│                  LangGraph create_agent              │
│         Middleware 链：                               │
│  FilesystemMiddleware → KnowledgeBaseMiddleware      │
│  → RuntimeConfigMiddleware → SkillsMiddleware        │
│  → ModelRetryMiddleware → TodoListMiddleware         │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│               知识库层 (src/knowledge/)               │
│                                                      │
│  KnowledgeBaseManager (manager.py)                   │
│  KnowledgeBaseFactory (factory.py)                   │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │  Milvus  │  │ LightRAG │  │   Dify   │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│                                                      │
│  chunking/ │ indexing.py │ graphs/                   │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│               数据层 (src/storage/ + repositories/)  │
│                                                      │
│  PostgreSQL (业务数据)    Milvus (向量)               │
│  Neo4j (图谱)            MinIO (文件)                 │
│  SQLite (LangGraph checkpoint)                       │
│                                                      │
│  repositories/: 每个业务实体对应一个 Repository 类    │
└─────────────────────────────────────────────────────┘
```

## 核心目录职责

| 目录 | 职责 |
|------|------|
| `server/` | FastAPI 入口、路由、中间件 |
| `server/routers/` | API 路由定义（chat/knowledge/graph/agent 等） |
| `src/agents/` | 智能体实现（chatbot/deep_agent/reporter/skills） |
| `src/agents/common/` | 智能体基类、中间件、工具集 |
| `src/knowledge/` | 知识库管理（Milvus/LightRAG/Dify 实现） |
| `src/knowledge/chunking/` | 文档分块策略 |
| `src/knowledge/graphs/` | 知识图谱相关逻辑 |
| `src/services/` | 业务服务层（对话/任务/MCP/Skill 等） |
| `src/repositories/` | 数据访问层（每实体一个 Repository） |
| `src/models/` | LLM 模型封装（chat/embed/rerank） |
| `src/config/` | 应用配置（Pydantic + TOML 持久化） |
| `src/storage/postgres/` | PostgreSQL ORM 模型定义 |
| `src/storage/minio/` | MinIO 文件存储封装 |
| `src/plugins/` | 插件扩展机制 |
| `src/utils/` | 通用工具函数 |
| `web/src/views/` | 前端页面视图 |
| `web/src/apis/` | 前端 API 接口定义 |
| `web/src/components/` | 可复用 UI 组件 |
| `web/src/stores/` | Pinia 状态管理 |

## 智能体中间件链

智能体采用 LangGraph `create_agent` + Middleware 链模式，执行顺序：

```
请求进入
  → save_attachments_to_fs      # 附件注入提示词
  → FilesystemMiddleware        # 文件系统后端（MinIO/本地）
  → KnowledgeBaseMiddleware     # 动态注入知识库工具
  → RuntimeConfigMiddleware     # 运行时应用模型/工具/MCP/提示词配置
  → SkillsMiddleware            # Skills 提示词注入与依赖展开
  → ModelRetryMiddleware        # 模型调用失败重试
  → TodoListMiddleware          # Todo 列表管理
  → PatchToolCallsMiddleware    # 工具调用修复
响应输出
```

代码位置：`src/agents/chatbot/graph.py`

## 知识库处理流程

```
文件上传 → 解析（PyMuPDF/docling/OCR）
        → 分块（chunking/）
        → Embedding（embed_model）
        → 入库（Milvus 向量 / LightRAG 图谱）
        → 检索时：向量检索 + 可选 Rerank
```

## 数据流：对话请求

```
前端 SSE 请求
  → chat_router.py
  → chat_stream_service.py
  → agent_run_service.py
  → LangGraph Agent.stream()
  → 工具调用（知识库/MCP/网络搜索）
  → 流式响应返回前端
  → conversation_service.py 持久化对话
```

## 任务队列

异步任务（文档解析入库等）通过 `run_queue_service.py` + `run_worker.py` 实现，使用 PostgreSQL 存储任务状态。

代码位置：`src/services/run_queue_service.py`, `server/worker_main.py`
