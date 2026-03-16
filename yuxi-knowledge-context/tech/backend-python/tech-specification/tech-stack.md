# 技术栈说明

> 项目：yuxi-know v0.5.1 | 更新：2026-03-16

## 语言与运行时

| 项目 | 版本 |
|------|------|
| Python | >=3.12, <3.14 |
| 包管理 | uv |

## 核心框架

| 框架 | 版本 | 用途 |
|------|------|------|
| FastAPI | >=0.121 | HTTP API 服务框架 |
| LangGraph | >=1.0.1 | 智能体图编排框架（v1） |
| LangChain | >=1.2.0 | LLM 调用抽象层 |
| LightRAG-HKU | >=1.4.6 | 知识图谱 RAG 引擎 |
| LlamaIndex | >=0.14 | 文档解析与索引 |
| Uvicorn | - | ASGI 服务器 |

## 数据存储

| 存储 | 用途 | 代码位置 |
|------|------|---------|
| PostgreSQL | 业务数据（用户、对话、任务等） | `src/storage/postgres/` |
| SQLite | 轻量级本地存储（LangGraph checkpoint） | `langgraph-checkpoint-sqlite` |
| Milvus | 向量数据库（RAG Embedding 检索） | `src/knowledge/implementations/milvus.py` |
| Neo4j | 图数据库（知识图谱存储） | `neo4j>=5.28.1` |
| MinIO | 对象存储（文件/附件） | `src/storage/minio/` |

## AI / LLM 集成

| 集成 | 说明 |
|------|------|
| langchain-openai | OpenAI 兼容接口 |
| langchain-deepseek | DeepSeek 模型支持 |
| dashscope | 阿里云百炼（通义千问、BGE 等） |
| langchain-tavily | 网络搜索工具 |
| langchain-mcp-adapters | MCP 协议适配 |
| MCP | Model Context Protocol >=1.20 |

## 文档解析

| 库 | 用途 |
|----|------|
| PyMuPDF | PDF 解析 |
| docling | Office 文件解析（docx/xlsx/pptx） |
| opencv-python-headless | 图片处理 |
| markdownify | HTML 转 Markdown |
| readability-lxml | 网页正文提取 |

## 默认模型配置

> 代码位置：`src/config/app.py`

| 配置项 | 默认值 |
|--------|--------|
| default_model | siliconflow/Pro/deepseek-ai/DeepSeek-V3.2 |
| fast_model | siliconflow/Qwen/Qwen3.5-9B |
| embed_model | siliconflow/Pro/BAAI/bge-m3 |
| reranker | siliconflow/Pro/BAAI/bge-reranker-v2-m3 |

## 部署

| 工具 | 用途 |
|------|------|
| Docker Compose | 全栈容器化管理（开发 + 生产） |
| docker-compose.yml | 开发环境（热重载） |
| docker-compose.prod.yml | 生产环境 |
