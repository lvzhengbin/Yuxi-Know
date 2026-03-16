# 编码规范

> 项目：yuxi-know v0.5.1 | 更新：2026-03-16

## 命名约定

### Python 后端

| 类型 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `KnowledgeBaseManager`, `ChatbotAgent`, `BaseAgent` |
| 方法/函数 | snake_case | `get_by_id`, `stream_messages`, `load_metadata` |
| 变量 | snake_case | `kb_instances`, `work_dir`, `agent_config` |
| 常量 | UPPER_SNAKE_CASE | `RATE_LIMIT_MAX_ATTEMPTS`, `RATE_LIMIT_WINDOW_SECONDS` |
| 私有方法 | `_` 前缀 | `_load_metadata`, `_get_or_create_kb_instance` |
| 异步方法 | 无特殊前缀，但必须 `async def` | `async def get_all()`, `async def initialize()` |
| 文件名 | snake_case | `knowledge_base_repository.py`, `chat_stream_service.py` |

### 前端 Vue

| 类型 | 规范 | 示例 |
|------|------|------|
| 组件文件 | PascalCase | `AgentView.vue`, `DataBaseView.vue` |
| API 文件 | snake_case + `_api` 后缀 | `knowledge_api.js`, `agent_api.js` |
| 变量/函数 | camelCase | `agentConfig`, `loadKnowledge` |

## 代码组织模式

### Repository 模式

数据访问层统一使用 Repository 类，每个业务实体对应一个文件。

```python
# ✅ 正确：通过 Repository 访问数据库
class KnowledgeBaseRepository:
    async def get_by_id(self, db_id: str) -> KnowledgeBase | None:
        async with pg_manager.get_async_session_context() as session:
            result = await session.execute(
                select(KnowledgeBase).where(KnowledgeBase.db_id == db_id)
            )
            return result.scalar_one_or_none()

# ❌ 错误：在 Service 或 Agent 中直接操作数据库 session
```

代码位置：`src/repositories/`

### BaseAgent 继承模式

所有智能体必须继承 `BaseAgent`，实现 `get_graph()` 方法。

```python
# ✅ 正确
class ChatbotAgent(BaseAgent):
    name = "智能体助手"
    description = "..."
    capabilities = ["file_upload", "files", "todo"]

    async def get_graph(self, **kwargs):
        graph = create_agent(
            model=load_chat_model(...),
            middleware=[...],
            checkpointer=await self._get_checkpointer(),
        )
        return graph

# ❌ 错误：直接在 Agent 中构建 LangGraph StateGraph，不使用 create_agent
```

代码位置：`src/agents/common/base.py`

### 中间件链模式

智能体功能扩展通过 Middleware 实现，不在 Agent 类中堆砌逻辑。

```python
# ✅ 正确：通过 middleware 列表组合功能
graph = create_agent(
    model=...,
    middleware=[
        KnowledgeBaseMiddleware(),
        RuntimeConfigMiddleware(extra_tools=all_mcp_tools),
        SkillsMiddleware(),
    ],
)

# ❌ 错误：在 get_graph() 中直接写工具注入、配置读取等逻辑
```

### 异步优先

所有 I/O 操作（数据库、文件、网络）必须使用 async/await。

```python
# ✅ 正确
async def get_all(self) -> list[KnowledgeBase]:
    async with pg_manager.get_async_session_context() as session:
        result = await session.execute(select(KnowledgeBase))
        return list(result.scalars().all())

# ❌ 错误：使用同步 ORM 操作阻塞事件循环
```

### 类型注解

Python 3.12+ 语法，使用新式类型注解。

```python
# ✅ 正确（Python 3.10+ union 语法）
def get_by_id(self, db_id: str) -> KnowledgeBase | None: ...

# ❌ 错误（旧式）
def get_by_id(self, db_id: str) -> Optional[KnowledgeBase]: ...
```

## 错误处理

- 业务异常使用自定义异常类，如 `KBNotFoundError`（`src/knowledge/base.py`）
- Router 层捕获异常并返回标准 HTTP 响应
- 底层服务层抛出异常，不吞掉错误

```python
# ✅ 正确：抛出具体业务异常
from src.knowledge.base import KBNotFoundError
raise KBNotFoundError(f"知识库 {db_id} 不存在")

# ❌ 错误：返回 None 或空字典掩盖错误
```

## 配置管理

应用配置统一通过 `src/config/app.py` 的 `Config` 类（Pydantic BaseModel）管理，支持 TOML 持久化。

```python
# ✅ 正确：从 config 读取配置
from src import config as sys_config
work_dir = Path(sys_config.save_dir) / "agents"

# ❌ 错误：硬编码路径或直接读取环境变量
```

## 前端规范

- 所有 API 接口定义在 `web/src/apis/` 下，按模块分文件
- 样式使用 Less，颜色必须使用 `base.css` 中的 CSS 变量
- Icon 优先使用 `lucide-vue-next`，其次 `@ant-design/icons-vue`
- UI 风格简洁，禁止悬停位移、过度阴影和渐变色
- 提交前运行 `npm run format` 格式化代码
