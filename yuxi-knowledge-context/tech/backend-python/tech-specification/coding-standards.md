# 编码规范 — backend-python

## 命名约定

| 类型 | 规则 | 示例 |
|------|------|------|
| 类 | PascalCase | `KnowledgeBaseManager`, `BaseAgent`, `ChatbotAgent` |
| 函数 / 方法 | snake_case | `stream_messages()`, `get_graph()`, `_get_checkpointer()` |
| 变量 | snake_case | `kb_instances`, `agent_config`, `work_dir` |
| 常量 | UPPER_SNAKE_CASE | `RATE_LIMIT_MAX_ATTEMPTS`, `RATE_LIMIT_WINDOW_SECONDS` |
| 私有方法 / 属性 | 单下划线前缀 | `_load_user_config()`, `_metadata_cache`, `_async_conn` |
| 抽象方法 | 同上，配合 `@abstractmethod` | `get_graph()` |
| 模块文件 | snake_case + 业务后缀 | `chat_stream_service.py`, `knowledge_router.py` |

---

## 代码组织模式

### 分层组织
- `server/routers/`：每个业务域一个 router 文件，只做 HTTP 协议处理
- `src/services/`：编排多个 Repository / Agent，不直接操作 ORM
- `src/repositories/`：数据访问，直接操作 SQLAlchemy ORM
- `src/agents/common/`：共享基础设施，Agent 实现继承 `BaseAgent`

### 抽象基类模式
```python
# 代码位置: src/agents/common/base.py
class BaseAgent:
    name = "base_agent"
    context_schema: type[BaseContext] = BaseContext

    @abstractmethod
    async def get_graph(self, **kwargs) -> CompiledStateGraph:
        pass
```

### 工厂模式
- `src/knowledge/factory.py`：根据 `kb_type` 实例化不同知识库
- `src/plugins/document_processor_factory.py`：根据文件类型分发解析器

### Singleton 模式（谨慎使用）
- 应用配置 `config = Config()`（`src/config/app.py` 模块级单例）
- `server/utils/singleton.py` 提供通用 Singleton 基类

---

## 异步规范

全项目统一使用 `async/await`，禁止在异步上下文中调用同步阻塞 IO。

```python
# ✅ 正确：异步 ORM
async def get_all(self) -> list[KnowledgeBase]:
    async with async_session() as session:
        result = await session.execute(select(KnowledgeBase))
        return result.scalars().all()

# ❌ 错误：在 async 函数中同步阻塞
def get_all(self):
    session = Session()
    return session.query(KnowledgeBase).all()
```

---

## 错误处理

### 自定义异常
```python
# 代码位置: src/knowledge/base.py
class KBNotFoundError(Exception): ...
```

### 统一 FastAPI 异常响应
- Router 层通过 `HTTPException` 返回标准 JSON 错误
- Service 层抛出业务异常，Router 层捕获并转换

### 重试
- 外部调用（LLM、向量数据库）使用 `tenacity` 重试装饰器
- LangGraph 递归限制: `recursion_limit=300`（chat）, `100`（invoke）

---

## 日志规范

项目使用 **Loguru** 作为主要日志库，通过 `src/utils/logging_config.py` 统一配置。

```python
from src.utils.logging_config import logger  # ✅ 统一导入路径
# 或
from src.utils import logger

logger.info("KnowledgeBaseManager initialized")
logger.debug(f"stream_messages: {context}")
logger.warning(f"POSTGRES_URL 未配置，回退 sqlite")
logger.error(f"获取智能体 {self.name} 历史消息出错: {e}")
```

**日志级别使用原则**：
- `DEBUG`：调试信息，详细上下文（context 内容等）
- `INFO`：关键流程节点（初始化完成、配置加载等）
- `WARNING`：降级处理、配置缺失（不影响功能）
- `ERROR`：异常捕获、功能失败

---

## 配置管理规范

- 所有配置通过 `src/config/app.py` 的 `Config` 类访问
- 环境变量通过 `python-dotenv` 加载（`.env` 文件）
- 用户修改的配置持久化到 `saves/config/base.toml`
- 禁止在代码中硬编码模型名称或 API Key

```python
from src import config as conf   # ✅
# 或
from src.config.app import config  # ✅

model = conf.default_model  # ✅ 属性访问
model = conf["default_model"]  # ⚠️ 已废弃，仅兼容旧代码
```

---

## 反模式清单

### 反模式 1：同步阻塞 IO
❌ 在 async 函数中使用同步文件读写或数据库操作
✅ 使用 `aiofiles`、`asyncpg`、`aiosqlite`
**原因**: 阻塞 uvicorn 事件循环，导致所有请求卡顿

### 反模式 2：直接在 Router 中编写业务逻辑
❌ `@router.post("/chat")` 内直接调用 ORM 查询、LLM 调用
✅ Router 只做参数解析和响应封装，业务逻辑放到 Service 层
**原因**: 难以测试，代码无法复用

### 反模式 3：不用 Repository 直接操作 ORM
❌ Service 层直接 `async with session: session.execute(...)`
✅ 通过 Repository 类封装数据访问
**原因**: 破坏分层，Service 层耦合数据库细节

### 反模式 4：字典式访问 Config
❌ `config["default_model"]`（会触发 deprecation warning）
✅ `config.default_model`（属性访问）
**原因**: 旧兼容接口，新代码应使用属性访问

### 反模式 5：MCP 超时未设置
❌ MCP HTTP 服务器连接不设置超时
✅ 连接时显式设置 timeout，避免 API 无限挂起
**原因**: 见 `experience/pitfalls/mcp-http-server-timeout-api-hang.md`

---

## Python 版本特性使用

项目目标 Python 3.12+，鼓励使用新语法：

```python
# ✅ 使用 | 替代 Optional / Union
def get_config(self) -> dict | None: ...

# ✅ 使用 match-case（Python 3.10+）
match kb_type:
    case "milvus": return MilvusKnowledgeBase(...)
    case "lightrag": return LightRAGKnowledgeBase(...)

# ✅ f-string 调试信息
logger.debug(f"{context=}")
```

*由 Agent Knowledge Kit v3.0 生成 · 2026-03-19*
