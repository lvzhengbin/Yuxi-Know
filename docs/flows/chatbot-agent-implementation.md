# Chatbot Agent 完整实现文档

## 概述

Yuxi-Know 的 Chatbot Agent (智能体助手) 是基于 **LangGraph v1** 构建的智能对话系统，融合了多层中间件架构、动态工具注入、知识库集成和技能管理等能力。本文档详细梳理其完整实现过程、对话调用流程以及与大模型的协议通讯机制。

---

## 1. 文件结构

```
src/agents/chatbot/
├── __init__.py          # 导出 ChatbotAgent
└── graph.py             # 主要实现

src/agents/common/
├── base.py              # BaseAgent 基类
├── context.py           # BaseContext 配置类
├── state.py             # BaseState 状态定义
├── models.py            # 模型加载 load_chat_model()
├── middlewares/
│   ├── runtime_config_middleware.py   # 运行时配置注入
│   ├── skills_middleware.py           # 技能管理
│   ├── knowledge_base_middleware.py   # 知识库工具
│   └── attachment_middleware.py       # 附件处理
├── toolkits/
│   ├── buildin/tools.py   # 内置工具
│   ├── kbs/tools.py       # 知识库工具
│   └── registry.py        # 工具注册机制
└── backends/
    └── composite.py       # 复合文件系统后端

server/routers/
└── chat_router.py         # API 路由

src/services/
└── chat_stream_service.py # 流式服务
```

---

## 2. 类层次结构

### ChatbotAgent 类定义

**位置**: `src/agents/chatbot/graph.py:25-62`

```python
class ChatbotAgent(BaseAgent):
    name = "智能体助手"
    description = "基础的对话机器人，可以回答问题，可在配置中启用需要的工具。"
    capabilities = ["file_upload", "files", "todo"]
```

继承自 `BaseAgent`，`BaseAgent` 提供：
- 图生命周期管理（构建、缓存、热重载）
- 历史记录管理（读写 Checkpoint 数据库）
- 通用流式接口 `stream_messages()`
- 智能体状态提取 `extract_agent_state()`

### BaseContext 配置类

**位置**: `src/agents/common/context.py:16-212`

```python
@dataclass
class BaseContext:
    model: str          # LLM 模型（如 "openai/gpt-4"）
    system_prompt: str  # 系统提示词
    tools: list[str]    # 启用的工具名列表
    knowledges: list    # 知识库配置
    mcps: list          # MCP 服务器配置
    skills: list        # 技能配置
```

---

## 3. 完整对话调用流程

```
用户 HTTP 请求
    │
    ▼
POST /chat/agent/{agent_id}
  [chat_router.py chat_agent(), 366-408行]
    │
    ▼
stream_agent_chat()
  [chat_stream_service.py, 328-551行]
    │ 1. 构建 HumanMessage（支持文字+图片）
    │ 2. 内容安全校验
    │ 3. 保存用户消息到 DB
    │
    ▼
AgentManager.get_agent(agent_id)
    │ 根据 agent_id 获取 ChatbotAgent 实例
    │
    ▼
agent.stream_messages(messages, input_context)
  [base.py, 80-101行]
    │ 1. 从 context_schema() 创建 Context
    │ 2. 用 input_context 更新 Context
    │ 3. 构建 langgraph_config（thread_id, user_id）
    │ 4. 调用 graph.astream(messages, stream_mode="messages", ...)
    │
    ▼
LangGraph 状态机执行
    │
    ├── 中间件 abefore_agent() 钩子依次执行
    │     1. save_attachments_to_fs   ← 附件内容注入 state
    │     2. FilesystemMiddleware     ← 文件系统后端初始化
    │     3. KnowledgeBaseMiddleware  ← 注册 KB 工具
    │     4. RuntimeConfigMiddleware  ← 注入模型/工具/系统提示词
    │     5. SkillsMiddleware         ← 注入技能提示词
    │     6. ModelRetryMiddleware
    │     7. TodoListMiddleware
    │     8. PatchToolCallsMiddleware
    │
    ▼
LLM 决策节点（Agent Node）
    │ 输入：消息历史 + 系统提示词 + 可用工具列表
    │ 输出：文本 token 流 或 工具调用请求
    │
    ├── 若有工具调用 ──▶ 工具执行节点（Tool Node）
    │                        │ 执行工具，返回 ToolMessage
    │                        └──▶ 回到 LLM 决策节点（循环）
    │
    └── 若无工具调用 ──▶ END
    │
    ▼
chat_stream_service.py 处理流式输出
    │ - AIMessageChunk  → 逐 token 下发，记录 TTFT
    │ - ToolMessage     → 下发工具调用事件
    │ - 检测 interrupt  → 触发人机交互中断
    │
    ▼
保存最终状态 + 发送 "finished" 状态
    │
    ▼
客户端接收 JSON Lines 流式响应
```

---

## 4. LangGraph 图结构

### 图构建

**位置**: `src/agents/chatbot/graph.py:33-62`

```python
async def get_graph(self, **kwargs):
    context = kwargs.get("context")
    graph = create_agent(
        model=load_chat_model(fully_specified_name=context.model),
        system_prompt=context.system_prompt,
        middleware=[...],  # 8个中间件
        checkpointer=await self._get_checkpointer(),
    )
    return graph
```

### 图节点与边

```
┌─────────────────────────────────────┐
│           LangGraph StateGraph      │
│                                     │
│  START                              │
│    │                                │
│    ▼                                │
│  [Agent Node]  ◄──────────┐         │
│    │ LLM 决策              │         │
│    │                      │         │
│    ├── tool_calls? ──▶ [Tool Node]  │
│    │                      │         │
│    └── no tools ──▶ END   │         │
│                           │         │
│  ToolMessage 自动返回 ──────┘         │
└─────────────────────────────────────┘
```

### 状态结构

```python
@dataclass
class BaseState:
    messages: Annotated[Sequence[AnyMessage], add_messages]
    # 由中间件动态扩展：
    # - todos: list (TodoListMiddleware)
    # - files: dict (FilesystemMiddleware)
    # - activated_skills: list[str] (SkillsMiddleware)
```

### 检查点（Checkpointing）

- 默认后端：SQLite `AsyncSqliteSaver`
- 路径：`{workdir}/agents/{module_name}/aio_history.db`
- 作用：按 `thread_id` 持久化完整对话历史，支持多轮对话和中断恢复

---

## 5. 中间件详解

中间件按顺序执行，每个中间件有两个钩子：
- `abefore_agent()` - 在 Agent 节点执行前运行
- `awrap_model_call()` - 包装 LLM 调用（可修改 messages/tools/system_prompt）

### 中间件执行顺序

| 序号 | 中间件 | 职责 |
|------|--------|------|
| 1 | `save_attachments_to_fs` | 将上传文件内容写入 `/attachments/` 虚拟路径 |
| 2 | `FilesystemMiddleware` | 提供 StateBackend + SkillsBackend 文件系统后端 |
| 3 | `KnowledgeBaseMiddleware` | 注入知识库工具（list_kbs、get_mindmap、query_kb） |
| 4 | `RuntimeConfigMiddleware` | 从 context 加载模型、合并工具列表、注入时间戳到系统提示词 |
| 5 | `SkillsMiddleware` | 注入技能提示词、展开依赖、动态激活技能 |
| 6 | `ModelRetryMiddleware` | LLM 调用失败重试 |
| 7 | `TodoListMiddleware` | 维护 Todo 列表状态 |
| 8 | `PatchToolCallsMiddleware` | 规范化工具调用格式 |

### RuntimeConfigMiddleware 详解

**位置**: `src/agents/common/middlewares/runtime_config_middleware.py:16-172`

```python
async def awrap_model_call(self, model_call, context, messages, ...):
    # 1. 从 context.model 加载 LLM
    model = load_chat_model(context.model)

    # 2. 从 context.tools 加载内置工具
    tools = get_all_tool_instances(context.tools)

    # 3. 从 context.mcps 加载 MCP 工具
    for mcp_server in context.mcps:
        tools += get_enabled_mcp_tools(mcp_server.name)

    # 4. 注入系统提示词（包含当前时间戳）
    system_prompt = f"{context.system_prompt}\n当前时间: {now()}"

    return await model_call(model, tools, system_prompt, messages)
```

### SkillsMiddleware 详解

**位置**: `src/agents/common/middlewares/skills_middleware.py:145-489`

- 将技能的 `SKILL.md` 内容注入为系统提示词扩展
- 展开技能依赖闭包（A 依赖 B 则同时激活 B）
- 检测 LLM 通过 `read_file` 工具访问 `/skills/` 路径来动态激活技能
- 将技能所需的额外工具/MCP 服务器合并到工具列表

---

## 6. 与大模型的协议通讯

### 模型加载

**位置**: `src/agents/common/models.py:12-57`

```python
def load_chat_model(fully_specified_name: str) -> BaseChatModel:
    provider, model_name = fully_specified_name.split("/", 1)

    match provider:
        case "openai":
            return init_chat_model(f"openai:{model_name}")
        case "deepseek":
            return init_chat_model(f"deepseek:{model_name}")
        case "dashscope":
            # 使用阿里云 DashScope，base_url 替换
            return ChatDeepSeek(model=model_name, base_url=DASHSCOPE_URL)
        case _:
            # 兼容 OpenAI 协议的其他提供商
            return ChatOpenAI(model=model_name, base_url=provider_url)
```

默认模型：`siliconflow/Pro/deepseek-ai/DeepSeek-V3.2`

### 流式通讯模式

LangGraph 使用 `stream_mode="messages"` 模式，返回 `(msg, metadata)` 元组：

```python
async for msg, metadata in graph.astream(
    input={"messages": [human_message]},
    stream_mode="messages",
    config={"configurable": {"thread_id": thread_id, "user_id": user_id}},
):
    if isinstance(msg, AIMessageChunk):
        # 逐 token 下发
        yield msg.content
    elif isinstance(msg, ToolMessage):
        # 工具执行结果
        yield tool_result
```

### 工具调用协议

1. **工具定义**（Pydantic Schema）:
```python
@tool(category="search", display_name="网页搜索")
def tavily_search(query: str) -> str:
    """搜索互联网获取最新信息"""
    ...
```

2. **LLM 返回工具调用**（OpenAI Function Calling 格式）:
```json
{
    "id": "call_abc123",
    "type": "tool_use",
    "name": "tavily_search",
    "args": {"query": "Python 最新版本"}
}
```

3. **工具执行结果**（ToolMessage）:
```json
{
    "type": "tool",
    "tool_call_id": "call_abc123",
    "content": "Python 3.13 于 2024年10月发布..."
}
```

4. **中断机制**（Human-in-the-Loop）：
   - 工具内部调用 `interrupt({"question": "...", "options": [...]})`
   - LangGraph 暂停图执行，保存状态
   - 客户端收到 `status: "ask_user_question_required"`
   - 用户通过 `/chat/agent/{id}/resume` 恢复执行

### TTFT（首 Token 延迟）监控

**位置**: `src/services/chat_stream_service.py:438-443`

```python
first_token_time = None
async for msg, metadata in agent.stream_messages(...):
    if first_token_time is None and isinstance(msg, AIMessageChunk):
        ttft = time.time() - start_time
        logger.info(f"[LLM TTFT] agent={agent_id} thread={thread_id} ttft={ttft:.3f}s")
        first_token_time = time.time()
```

---

## 7. 流式响应协议

### HTTP 响应格式

- **Content-Type**: `application/json`（JSON Lines 格式，每行一个 JSON 对象）
- **编码**: 每行末尾 `\n`

### 状态码与消息格式

```jsonc
// 初始化：保存用户消息
{"request_id": "...", "status": "init", "meta": {...}, "msg": {"role": "user", "content": "..."}}

// 流式 token
{"request_id": "...", "status": "loading", "response": "Python", "msg": {...}, "metadata": {...}}

// 工具调用事件
{"request_id": "...", "status": "loading", "response": null, "msg": {"type": "tool", ...}}

// 状态更新（todos/files）
{"request_id": "...", "status": "agent_state", "agent_state": {"todos": [], "files": {}}}

// 人机交互中断
{"request_id": "...", "status": "ask_user_question_required", "question_id": "...", "options": [...]}

// 完成
{"request_id": "...", "status": "finished", "meta": {"time_cost": 2.345}}

// 错误
{"request_id": "...", "status": "error", "msg": "错误信息"}
```

### 状态码说明

| status | 含义 |
|--------|------|
| `init` | 初始化，用户消息已保存 |
| `loading` | 正在流式输出 token 或工具消息 |
| `agent_state` | 智能体状态更新（todos/files 变化） |
| `ask_user_question_required` | 触发人机交互中断，等待用户回复 |
| `finished` | 对话完成 |
| `error` | 发生错误 |
| `interrupted` | 连接中断 |

---

## 8. API 端点

### 核心对话端点

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/chat/agent/{agent_id}` | 发送消息，流式返回响应 |
| `POST` | `/chat/agent/{agent_id}/resume` | 恢复中断的对话 |

### 配置与管理端点

| 方法 | 路径 | 说明 |
|------|------|------|
| `GET` | `/chat/agent/{agent_id}` | 获取智能体信息及可配置项 |
| `GET` | `/chat/agent/{agent_id}/config` | 读取已保存的配置 YAML |
| `POST` | `/chat/agent/{agent_id}/config` | 保存配置并热重载图 |
| `GET` | `/chat/agent/{agent_id}/history` | 获取对话历史 |
| `GET` | `/chat/agent/{agent_id}/state` | 获取当前状态（todos/files） |
| `GET` | `/chat/agent` | 列出所有可用智能体 |

### 请求示例

```json
POST /chat/agent/ChatbotAgent
{
    "query": "Python 是什么？",
    "config": {
        "thread_id": "thread-abc123",
        "agent_config_id": 1
    },
    "meta": {
        "request_id": "req-xyz789"
    },
    "image_content": null
}
```

---

## 9. 工具系统

### 内置工具

**位置**: `src/agents/common/toolkits/buildin/tools.py`

| 工具名 | 说明 |
|--------|------|
| `calculator` | 基础数学计算 |
| `ask_user_question` | 向用户提问（触发 interrupt） |
| `tavily_search` | 网页搜索（需配置 TAVILY_API_KEY） |

### 知识库工具

**位置**: `src/agents/common/toolkits/kbs/tools.py`

| 工具名 | 说明 |
|--------|------|
| `list_kbs` | 列出可访问的知识库 |
| `get_mindmap` | 获取知识库思维导图/结构 |
| `query_kb` | 检索知识库内容 |

### 工具注册方式

```python
from src.agents.common.toolkits.registry import tool

@tool(category="search", tags=["web"], display_name="网页搜索")
def my_search_tool(query: str) -> str:
    """搜索网页获取最新信息"""
    return results
```

### MCP 工具集成

MCP（Model Context Protocol）服务器工具通过 `src/services/mcp_service.py` 动态加载：

1. Agent 配置中指定 `mcps` 列表
2. `RuntimeConfigMiddleware` 在每次 LLM 调用前加载对应 MCP 服务器的工具
3. 工具与内置工具合并，去重后传给 LLM

---

## 10. 状态管理与内存

### 对话状态持久化

```
用户1 (user_id=u1)
├── 对话1 (thread_id=t1) ─── 多个 Checkpoint 快照
├── 对话2 (thread_id=t2) ─── 多个 Checkpoint 快照
└── ...

用户2 (user_id=u2)
└── ...
```

每个 Checkpoint 包含完整的 `messages` 列表，支持：
- 多轮对话上下文保持
- 从中断点恢复
- 历史记录查询

### 文件系统后端

`FilesystemMiddleware` 提供虚拟文件系统，路径映射：

| 虚拟路径 | 内容 | 后端 |
|----------|------|------|
| `/attachments/` | 用户上传的文件 | StateBackend（存于 LangGraph State） |
| `/skills/{slug}/SKILL.md` | 技能说明文档 | SkillsReadonlyBackend（只读） |

### 智能体状态（Agent State）

对话结束时通过 `extract_agent_state()` 提取：

```python
{
    "todos": [
        {"id": "...", "title": "...", "status": "pending"}
    ],
    "files": {
        "/attachments/report.pdf": {
            "content": ["第一页内容...", "第二页内容..."],
            "created_at": "...",
            "modified_at": "..."
        }
    }
}
```

---

## 11. 扩展指南

### 创建自定义 Agent

```python
from src.agents.common.base import BaseAgent

class MyCustomAgent(BaseAgent):
    name = "我的自定义助手"
    description = "专门处理某类任务的智能体"
    capabilities = ["file_upload", "files"]

    async def get_graph(self, **kwargs):
        context = kwargs.get("context")
        # 自定义中间件配置
        graph = create_agent(
            model=load_chat_model(context.model),
            middleware=[...],
            checkpointer=await self._get_checkpointer(),
        )
        return graph
```

将文件放在 `src/agents/{name}/` 目录下，`AgentManager` 会自动发现并注册。

### 添加自定义工具

```python
from src.agents.common.toolkits.registry import tool

@tool(category="custom", tags=["business"], display_name="业务查询")
def query_business_data(entity_id: str, field: str) -> dict:
    """查询业务系统数据

    Args:
        entity_id: 实体ID
        field: 查询字段名
    """
    return {...}
```

---

*文档生成时间: 2026-03-17*
