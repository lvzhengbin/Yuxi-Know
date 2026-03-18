# ChatbotAgent 长对话上下文管理

**状态**：待实施
**日期**：2026-03-17
**模块**：`src/agents/chatbot/graph.py`

---

## 问题描述

`ChatbotAgent`（智能聊天助手）目前没有配置上下文压缩机制，历史消息会随对话轮次无限累积后全量传给 LLM。当对话轮次较多时，消息总量会超出模型上下文窗口，导致报错或响应质量下降。

## 根因

`ChatbotAgent.get_graph()` 的中间件列表中未包含 `SummaryOffloadMiddleware`，而同项目的 `DeepAgent` 已配置了该中间件（触发阈值 90k tokens）。

```python
# 当前 chatbot/graph.py 中间件列表（无上下文压缩）
middleware=[
    save_attachments_to_fs,
    FilesystemMiddleware(backend=_create_fs_backend),
    KnowledgeBaseMiddleware(),
    RuntimeConfigMiddleware(extra_tools=all_mcp_tools),
    SkillsMiddleware(),
    ModelRetryMiddleware(),
    TodoListMiddleware(),
    PatchToolCallsMiddleware(),
]
```

## 需求

在 `ChatbotAgent` 的中间件列表中加入 `SummaryOffloadMiddleware`，实现超长对话的自动摘要压缩。

### 建议配置参数

参考 `DeepAgent` 的配置，结合 ChatbotAgent 使用场景（普通对话，上下文复杂度低于深度研究）适当降低触发阈值：

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `trigger` | `("tokens", 60000)` | 60k tokens 时触发（DeepAgent 为 90k） |
| `trim_tokens_to_summarize` | `4000` | 摘要时最多处理 4k tokens |
| `summary_offload_threshold` | `500` | 工具结果超过 500 tokens 时卸载到文件系统 |
| `max_retention_ratio` | `0.5` | 压缩后保留 50% 以内的 tokens |

### 实施位置

`src/agents/chatbot/graph.py` 的 `get_graph()` 方法：

```python
summary_middleware = SummaryOffloadMiddleware(
    model=model,
    trigger=("tokens", 60000),
    trim_tokens_to_summarize=4000,
    summary_offload_threshold=500,
    max_retention_ratio=0.5,
)

graph = create_agent(
    model=model,
    ...
    middleware=[
        ...,
        summary_middleware,  # 添加在列表末尾
    ],
)
```

注意：`model` 需要在 `get_graph()` 中提前通过 `load_chat_model(context.model)` 加载，而不是延迟到 `create_agent` 内部。

## 验收标准

- 普通对话（< 60k tokens）：行为与现在一致，无感知
- 超长对话（> 60k tokens）：自动触发摘要压缩，不报上下文溢出错误
- 压缩后历史仍保留关键信息，AI 能正确理解前文语境
