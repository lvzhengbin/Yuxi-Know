# Yuxi-Know 知识库索引

> 版本：v0.5.1 | 项目类型：backend-python | 最后更新：2026-03-16

## 项目简介

Yuxi-Know 是一个基于大模型的智能知识库与知识图谱智能体开发平台，融合了 RAG 技术与知识图谱技术，基于 LangGraph v1 + Vue.js + FastAPI + LightRAG 架构构建。

## 知识库结构

```
yuxi-knowledge-context/
├── INDEX.md                          # 本文件，知识库总索引
├── overview/                         # 项目全景
│   ├── project-overview.md           # 项目概述
│   └── architecture-overview.md      # 架构总览
├── business/                         # 业务知识
│   ├── domains/                      # 业务域
│   │   └── domain-catalog.md         # 业务域目录
│   ├── entities/                     # 核心实体
│   ├── flows/                        # 业务流程
│   ├── rules/                        # 业务规则
│   └── glossary/                     # 术语表
├── tech/
│   ├── backend-python/               # 后端技术知识
│   │   ├── tech-specification/       # 技术规格
│   │   │   ├── tech-stack.md         # 技术栈说明
│   │   │   └── architecture-overview.md
│   │   ├── standards/                # 开发规范
│   │   └── experience/               # 经验沉淀
│   │       ├── incidents/            # 故障复盘
│   │       ├── pitfalls/             # 踩坑记录
│   │       ├── performance/          # 性能优化
│   │       ├── tech-debt/            # 技术债务
│   │       └── best-practices/       # 最佳实践
│   └── frontend-vue/                 # 前端技术知识
│       ├── tech-specification/
│       ├── standards/
│       └── experience/
│           ├── incidents/
│           ├── pitfalls/
│           ├── performance/
│           ├── tech-debt/
│           └── best-practices/
└── operations/                       # 运维知识
```

## 快速导航

| 分类 | 文件 | 说明 |
|------|------|------|
| 项目全景 | [overview/project-overview.md](overview/project-overview.md) | 项目概述与背景 |
| 架构总览 | [overview/architecture-overview.md](overview/architecture-overview.md) | 系统架构图与说明 |
| 业务域 | [business/domains/domain-catalog.md](business/domains/domain-catalog.md) | 业务域目录 |
| 后端技术栈 | [tech/backend-python/tech-specification/tech-stack.md](tech/backend-python/tech-specification/tech-stack.md) | Python 后端技术栈 |
| 前端技术栈 | [tech/frontend-vue/tech-specification/tech-stack.md](tech/frontend-vue/tech-specification/tech-stack.md) | Vue 前端技术栈 |

## 经验沉淀

### 坑点记录（Pitfalls）

| 文件 | 关键词 | 说明 |
|------|--------|------|
| [tech/backend-python/experience/pitfalls/venv-detection-sys-prefix-vs-executable.md](tech/backend-python/experience/pitfalls/venv-detection-sys-prefix-vs-executable.md) | venv、sys.prefix、os.execv、无限循环、macOS Python 3.13 | Python venv 检测应用 sys.prefix 而非 sys.executable，否则在 macOS+Python 3.13 上会导致无限循环 |

## 服务列表

| 服务名 | 技术栈 | 路径 | 职责 |
|--------|--------|------|------|
| service-backend | Python / FastAPI + LangGraph + LightRAG | src/ + server/ | RAG 知识库后端 + 智能体 API 服务 |
| web | JavaScript / Vue.js + Vite | web/ | 前端管理界面 |
