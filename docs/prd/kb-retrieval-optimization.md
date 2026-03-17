# 知识库检索行为优化

**状态**：待实施
**日期**：2026-03-17
**模块**：`src/agents/common/middlewares/knowledge_base_middleware.py`

---

## 问题描述

在智能聊天助手中，当 Agent 配置了知识库时，即使用户提问与知识库内容无关（如"解释一下什么是机器学习"），Agent 仍会调用 `list_kbs → query_kb` 等知识库工具，造成无效检索。

## 根因

1. **工具描述缺乏使用边界**：`list_kbs` / `query_kb` 的 docstring 只描述工具能做什么，没有说明何时**不该**调用，LLM 倾向于"尽职尽责"地先查资源再回答。

2. **系统提示词无约束**：默认 `system_prompt = "You are a helpful assistant."` 未指导 LLM 何时跳过知识库直接回答。

3. **`KnowledgeBaseMiddleware` 盲注入工具**：只要 Agent 配置了知识库，每次 LLM 调用都无条件注入 3 个知识库工具，无任何相关性过滤。

---

## 优化方案

### 方案 A — 优化工具描述 + 注入使用约束提示词（快速）

在 `KnowledgeBaseMiddleware.awrap_model_call` 中注入知识库使用原则到系统提示词：

```
知识库使用原则：
- 仅当用户的问题明确需要参考知识库中的特定内容时，才调用知识库工具
- 对于通用概念解释、常识、编程基础等问题，直接回答，不需要调用知识库
- 判断标准：这个问题的答案"只存在于知识库中"还是"通用知识就能回答"？
```

同时在 `list_kbs` docstring 补充：
```
注意：仅在用户问题可能与知识库内容相关时才调用此工具。
```

- **改动**：`knowledge_base_middleware.py` + `kbs/tools.py`，约 10 行
- **效果**：70~80% 改善
- **缺点**：依赖 LLM 遵守指令

### 方案 B — 基于知识库描述做相关性预判（推荐长期方案）

在注入工具前，取用户最新消息与各知识库 `description` 做关键词/语义匹配。若无相关知识库，则本次 LLM 调用不注入知识库工具。

```
用户问题 → 与各 KB description 匹配
         → 相关性低 → 不注入 KB 工具（LLM 直接回答）
         → 相关性高 → 正常注入工具
```

- **改动**：`KnowledgeBaseMiddleware.awrap_model_call` 中增加预判逻辑
- **效果**：90%+ 改善
- **缺点**：KB description 质量影响召回率；需实现轻量匹配逻辑

### 方案 C — 两阶段工具设计（彻底方案）

拆分为探索 + 检索两阶段：先暴露一个 `should_query_kb(question)` 判断工具，由 LLM 决策后再提供 `query_kb`。

- **改动**：架构调整，改动较大
- **效果**：接近 100%
- **缺点**：每次对话多一次 LLM 调用，延迟增加

---

## 建议实施顺序

**A + B 组合**：A 作为 LLM 行为约束兜底，B 作为工具注入前的过滤门槛，两者互补。

| 方案 | 改动量 | 效果 | 优先级 |
|------|--------|------|--------|
| A    | 极小   | 70~80% | P1（先做）|
| B    | 中等   | 90%+   | P1（同步推进）|
| C    | 较大   | ~100%  | P2（视效果决定）|

## 涉及文件

- `src/agents/common/middlewares/knowledge_base_middleware.py`
- `src/agents/common/toolkits/kbs/tools.py`
