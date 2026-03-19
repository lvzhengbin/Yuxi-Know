# 技术栈清单 — backend-python

**项目类型**: Python 后端服务
**语言版本**: Python 3.12+（`requires-python = ">=3.12,<3.14"`）
**包管理工具**: uv（清华源加速）
**代码规范工具**: Ruff（行宽 120）

---

## 核心框架

| 技术 | 版本 | 用途 |
|------|------|------|
| FastAPI | >=0.121 | Web API 框架，提供 REST 接口和 SSE 流式输出 |
| uvicorn | >=0.34.2 | ASGI 服务器，支持 standard 模式 |
| LangGraph | >=1.0.1 | Agent 编排框架，基于状态机构建多智能体 |
| LangChain | >=1.2.0 | LLM 抽象层，提供 Chain/Tool/Memory 等接口 |
| LightRAG-HKU | >=1.4.6 | 知识图谱 RAG 框架，支持图谱构建与检索 |
| Pydantic | (随 FastAPI) | 数据验证与配置管理 |

## 数据存储

| 技术 | 版本 | 用途 |
|------|------|------|
| SQLAlchemy[asyncio] | >=2.0.0 | 异步 ORM，管理业务数据模型 |
| asyncpg | >=0.30.0 | PostgreSQL 异步驱动 |
| aiosqlite | >=0.20.0 | SQLite 异步驱动（LangGraph checkpointer 默认） |
| pymilvus | >=2.5.8 | Milvus 向量数据库客户端（RAG 向量检索） |
| neo4j | >=5.28.1 | Neo4j 图数据库（知识图谱存储） |
| networkx | >=3.5 | 内存图计算（图谱分析辅助） |
| redis | >=5.2.0 | 缓存 / 任务队列后端 |

## 任务队列

| 技术 | 版本 | 用途 |
|------|------|------|
| arq | >=0.26.3 | 基于 Redis 的异步任务队列（文档入库等耗时任务） |

## 对象存储

| 技术 | 版本 | 用途 |
|------|------|------|
| minio | >=7.2.7 | MinIO / S3 兼容对象存储（文件上传持久化） |

## LLM 模型集成

| 技术 | 版本 | 用途 |
|------|------|------|
| langchain-openai | >=1.0.2 | OpenAI 兼容模型（SiliconFlow 等） |
| langchain-deepseek | >=1.0 | DeepSeek 模型 |
| dashscope | >=1.23.2 | 阿里云百炼（通义系列） |
| langchain-mcp-adapters | >=0.1.9 | MCP（Model Context Protocol）工具适配 |
| langchain-tavily | >=0.2.13 | Tavily 网络搜索工具 |
| openai | >=1.109 | OpenAI SDK 直接调用 |
| deepagents | >=0.2.5 | DeepAgents 深度智能体 |

## 文档解析

| 技术 | 版本 | 用途 |
|------|------|------|
| docling | >=2.68.0 | Office 文档（docx/xlsx/pptx）解析 |
| pymupdf | >=1.25.5 | PDF 解析 |
| rapidocr-onnxruntime | >=1.4.4 | 本地 OCR（RapidOCR） |
| opencv-python-headless | >=4.11.0.86 | 图像预处理 |
| Pillow | >=10.5.0 | 图像处理 |
| torch / torchvision | >=2.8.0 / ==0.23 | 深度学习推理（CPU 版） |
| unstructured | >=0.17.2 | 通用文档解析 |
| llama-index | >=0.14 | 文档分块与索引辅助 |

## 认证与安全

| 技术 | 版本 | 用途 |
|------|------|------|
| pyjwt | >=2.8.0 | JWT 令牌生成 |
| python-jose[cryptography] | >=3.4.0 | JWT 验证 |
| python-multipart | >=0.0.20 | 文件上传表单解析 |

## 工具类

| 技术 | 版本 | 用途 |
|------|------|------|
| loguru | >=0.7.3 | 结构化日志（主要日志库） |
| colorlog | >=6.9.0 | 控制台彩色日志 |
| tenacity | >=8.0.0 | 重试装饰器 |
| python-dotenv | >=1.1.0 | .env 环境变量加载 |
| httpx | >=0.27.0 | 异步 HTTP 客户端 |
| aiohttp | >=3.9.0 | 异步 HTTP 客户端（备用） |
| pyyaml | >=6.0.2 | YAML 配置解析 |
| tomli / tomli-w | - | TOML 配置读写（应用配置持久化） |
| rich | >=13.7.1 | 终端富文本输出 |
| typer | >=0.16.0 | CLI 命令行工具 |
| json-repair | >=0.54.0 | 修复 LLM 输出的格式错误 JSON |

## 开发与测试

| 技术 | 版本 | 用途 |
|------|------|------|
| ruff | >=0.12.1 | Lint + 格式化（替代 flake8 + isort） |
| pytest | >=8.0.0 | 测试框架 |
| pytest-asyncio | >=0.24.0 | 异步测试支持 |
| pytest-httpx | >=0.32.0 | 模拟 httpx 请求 |
| pytest-cov | >=6.0.0 | 测试覆盖率 |

---

## 关键技术决策

- **为什么选 LangGraph？** 支持有状态的多节点 Agent 图，原生 checkpointer 持久化对话历史，适合复杂多步骤推理。
- **为什么 SQLite + PostgreSQL 双模式？** 开发环境默认 SQLite 零依赖，生产环境通过 `LANGGRAPH_CHECKPOINTER_BACKEND=postgres` 切换。
- **为什么 Milvus + LightRAG 并存？** Milvus 负责向量检索（Dense RAG），LightRAG 负责知识图谱 RAG，两者互补。
- **为什么 uv？** 相比 pip/poetry 显著提速依赖安装，lockfile 确保可复现构建。

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
