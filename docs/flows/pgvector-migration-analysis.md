# 技术可行性评估：PostgreSQL + pgvector 替换 Milvus

> 分析日期：2026-03-23
> 前提：新系统改造，不涉及历史数据迁移
> 分析范围：`src/knowledge/`、`docker-compose.yml`、`src/storage/postgres/`

---

## 结论先行

| 组件 | 可行性 | 工作量 | 风险 |
|------|--------|--------|------|
| **MilvusKB → PostgresKB** | ✅ 完全可行 | 低（~5天） | 低 |
| **LightRagKB 中的 Milvus** | ⚠️ 条件可行 | 中（~3天） | 中 |
| **基础设施改造** | ✅ 极简改动 | 极低（1小时） | 无 |

---

## 一、MilvusKB 替换分析

### 1.1 Milvus 特性逐项评估

读完 `src/knowledge/implementations/milvus.py` 后，所有 Milvus 特性都有 pgvector 等价实现：

| Milvus 用法 | 代码位置 | pgvector 等价方案 | 难度 |
|------------|----------|-----------------|------|
| `FLOAT_VECTOR(dim)` schema 字段 | `_create_new_collection` L148 | `vector(dim)` 列类型 | 无 |
| `IVF_FLAT` + `COSINE` 索引 | `_create_new_collection` L159 | `USING ivfflat (embedding vector_cosine_ops)` | 无 |
| `collection.load()` 预加载 | `_initialize_kb_instance` L169 | 不需要，PG 自动缓存 | 无 |
| `collection.search()` ANN 向量检索 | `aquery` L520 | `ORDER BY embedding <=> $vec LIMIT k` | 低 |
| `collection.query(expr=...)` 标量过滤 | `aquery` L569，`get_file_content` L735 | SQL `WHERE file_id=? AND content LIKE ?` | 无 |
| `collection.insert(entities)` 批量写入 | `index_file` L330 | `INSERT INTO ... VALUES (...)` 批量 | 低 |
| `collection.delete(expr)` 按条件删除 | `delete_file_chunks_only` L684 | `DELETE WHERE file_id=?` | 无 |
| `utility.has_collection()` | `_create_kb_instance` L106，`delete_database` L786 | `SELECT to_regclass(...)` | 无 |
| `utility.drop_collection()` | `delete_database` L787 | `DELETE WHERE db_id=?`（单表多库，无需 drop） | 低 |

**关键发现**：`aquery` 中的核心业务逻辑——hybrid 融合（L607-L632）、reranker 重排序（L648-L672）——**完全在应用层实现，与存储引擎无关**，迁移后无需任何改动。

### 1.2 表结构设计

Milvus 每个 `db_id` 对应一个独立 Collection，迁移后改为**单表多 `db_id`** 模式，在现有 `models_knowledge.py` 中追加：

```python
from pgvector.sqlalchemy import Vector

class KnowledgeChunk(Base):
    __tablename__ = "knowledge_chunks"

    id          = Column(BigInteger, primary_key=True, autoincrement=True)
    chunk_id    = Column(String(100), unique=True, nullable=False, index=True)
    db_id       = Column(String(80), ForeignKey("knowledge_bases.db_id", ondelete="CASCADE"), nullable=False)
    file_id     = Column(String(100), ForeignKey("knowledge_files.file_id", ondelete="CASCADE"), nullable=False)
    content     = Column(Text, nullable=False)
    source      = Column(String(500), index=True)
    chunk_index = Column(Integer, nullable=False)
    embedding   = Column(Vector(1024))   # 维度随 embed_info["dimension"] 动态配置

    __table_args__ = (
        Index("idx_chunks_db_id", "db_id"),
        Index("idx_chunks_file_id", "file_id"),
        Index("idx_chunks_embedding", "embedding",
              postgresql_using="ivfflat",
              postgresql_with={"lists": 100},
              postgresql_ops={"embedding": "vector_cosine_ops"}),
    )
```

对应 DDL：
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE knowledge_chunks (
    id          BIGSERIAL PRIMARY KEY,
    chunk_id    VARCHAR(100) UNIQUE NOT NULL,
    db_id       VARCHAR(80) NOT NULL REFERENCES knowledge_bases(db_id) ON DELETE CASCADE,
    file_id     VARCHAR(100) NOT NULL REFERENCES knowledge_files(file_id) ON DELETE CASCADE,
    content     TEXT NOT NULL,
    source      VARCHAR(500),
    chunk_index INT NOT NULL,
    embedding   vector(1024)
);

CREATE INDEX ON knowledge_chunks (db_id);
CREATE INDEX ON knowledge_chunks (file_id);
CREATE INDEX ON knowledge_chunks
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

> **向量维度说明**：pgvector 要求建表时固定维度，而现有代码中维度来自运行时的 `embed_info["dimension"]`。
> 新系统场景下，建库时 embed_info 已确定，可在 `_create_kb_instance` 阶段动态建表或用最大维度（1024）统一处理。

### 1.3 核心方法改写对照

**向量检索（`aquery` 向量部分）：**

```python
# Milvus 方式（现状）
results = collection.search(
    data=query_embedding,
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"nprobe": 10}},
    limit=recall_top_k,
    expr=file_expr,
    output_fields=["content", "source", "chunk_id", "file_id", "chunk_index"],
)

# pgvector 方式
async with session() as s:
    rows = await s.execute(
        select(
            KnowledgeChunk.content, KnowledgeChunk.source,
            KnowledgeChunk.chunk_id, KnowledgeChunk.file_id, KnowledgeChunk.chunk_index,
            KnowledgeChunk.embedding.cosine_distance(query_embedding).label("distance"),
        )
        .where(KnowledgeChunk.db_id == db_id)
        .where(KnowledgeChunk.source.like(f"%{file_name}%"))   # 可选过滤
        .order_by("distance")
        .limit(recall_top_k)
    )
```

**关键词检索（`aquery` keyword 部分）：**

```python
# Milvus 方式（现状）
keyword_expr = " or ".join([f'content like "%{kw}%"' for kw in keywords])
results = collection.query(expr=keyword_expr, limit=keyword_top_k)

# pgvector 方式（直接使用 SQL）
rows = await s.execute(
    select(KnowledgeChunk)
    .where(or_(*[KnowledgeChunk.content.ilike(f"%{kw}%") for kw in keywords]))
    .where(KnowledgeChunk.db_id == db_id)
    .limit(keyword_top_k)
)
```

---

## 二、LightRagKB 的 Milvus 依赖

`src/knowledge/implementations/lightrag.py` 中有两处直接依赖：

### 2.1 依赖1：`delete_database` 直接调用 pymilvus（可直接改写）

```python
# 现状（lightrag.py L60-78）
from pymilvus import connections, utility
connections.connect(alias=connection_alias, uri=milvus_uri, token=milvus_token)
collection_names = [f"{db_id}_chunks", f"{db_id}_relationships", f"{db_id}_entities"]
for collection_name in collection_names:
    if utility.has_collection(collection_name, using=connection_alias):
        utility.drop_collection(collection_name, using=connection_alias)
```

改写为 pgvector 后只需：
```python
# 改写后（删除 LightRAG 在 PG 中写入的向量数据）
async with session() as s:
    await s.execute(
        delete(KnowledgeChunk).where(KnowledgeChunk.db_id.like(f"{db_id}%"))
    )
```

改动量：约 10 行，删除 pymilvus import。

### 2.2 依赖2：LightRAG 库的向量存储后端（需验证）

```python
# 现状
LightRAG(vector_storage="MilvusVectorDBStorage", ...)

# 目标
LightRAG(vector_storage="PGVectorStorage", ...)
```

`lightrag-hku >= 1.3` 已引入 `PGVectorStorage`，需**验证当前锁定版本（`>= 1.4.6`）是否可用**：

```bash
docker compose exec api uv run python -c "
import lightrag; print(lightrag.__version__)
from lightrag.storage import PGVectorStorage
print('PGVectorStorage: OK')
"
```

- **支持** → LightRagKB 完全去除 Milvus，整个系统只依赖 PostgreSQL
- **不支持** → LightRagKB 暂时保留 Milvus，仅 MilvusKB 替换（两套并存）

---

## 三、基础设施改动

### 3.1 docker-compose.yml（改 1 行）

```yaml
# 改前
postgres:
  image: postgres:16

# 改后（官方 pgvector 镜像，内置扩展）
postgres:
  image: pgvector/pgvector:pg16
```

PostgreSQL 启动后执行一次（可放入初始化脚本）：
```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

**现有的以下内容全部无需改动**：
- `PostgresManager`（`src/storage/postgres/manager.py`）
- SQLAlchemy 异步引擎（`asyncpg` 驱动）
- `JSONB` 支持
- `KnowledgeBaseRepository`、`KnowledgeFileRepository` 等 Repository 层
- `POSTGRES_URL` 环境变量配置

### 3.2 依赖变更（`pyproject.toml`）

```toml
# 新增
pgvector = ">=0.3.0"

# 待移除（确认 LightRAG 不再依赖后）
# pymilvus
```

---

## 四、改动文件清单

| 文件 | 操作 | 改动量 |
|------|------|--------|
| `docker-compose.yml` | 修改 | 1 行（image 替换） |
| `src/storage/postgres/models_knowledge.py` | 新增 | ~25 行（`KnowledgeChunk` 模型） |
| `src/knowledge/implementations/postgres.py` | 新建 | ~300 行（`PostgresKB` 实现） |
| `src/knowledge/__init__.py` | 修改 | 1-2 行（注册 `PostgresKB`） |
| `src/knowledge/implementations/lightrag.py` | 修改 | ~15 行（`delete_database` + `vector_storage` 参数） |
| `pyproject.toml` | 修改 | 新增 `pgvector`，视情况移除 `pymilvus` |

---

## 五、性能参考

| 场景 | Milvus | PostgreSQL + pgvector | 说明 |
|------|--------|-----------------------|------|
| 百万级向量 ANN 查询 | ~50ms | ~150-300ms | Milvus 专用优化更快 |
| 十万级以下查询 | ~50ms | ~50-100ms | 差距可忽略 |
| 复杂 WHERE 过滤 | 受限 | 完整 SQL，更灵活 | PG 更强 |
| 批量插入 | 快 | 略慢（可调优） | 可接受 |
| 内存占用 | 高（预加载） | 低（按需） | PG 更省 |
| 运维复杂度 | 高（milvus+etcd+minio） | 低（复用现有 PG） | PG 大幅降低 |

> 当前项目知识库规模通常在十万 chunk 以内，pgvector 性能完全满足需求。

---

## 六、实施建议

### 步骤一：验证 LightRAG 兼容性（1小时）
运行上方验证命令，确认 `PGVectorStorage` 是否可用，决定是否一次性彻底去除 Milvus。

### 步骤二：基础设施准备（1小时）
- `docker-compose.yml` 替换 image
- 执行 `CREATE EXTENSION IF NOT EXISTS vector`
- 新增 `KnowledgeChunk` 模型并迁移表结构

### 步骤三：实现 PostgresKB（3-4天）
继承 `KnowledgeBase`，按 Milvus 实现逐方法替换，重点：
- `_create_kb_instance`：建表/确认表存在
- `_initialize_kb_instance`：空实现（PG 无需预加载）
- `index_file`：embedding 批量插入
- `aquery`：向量检索 + 关键词检索 + 原有融合逻辑（复用）
- `delete_file` / `get_file_content`：SQL 改写

### 步骤四：改造 LightRagKB（1天）
- 改写 `delete_database`，移除 pymilvus 直接调用
- 若步骤一验证通过，将 `vector_storage` 切换为 `PGVectorStorage`

### 步骤五：注册与测试（1天）
- `__init__.py` 注册 `PostgresKB`
- 联调测试：上传 → 解析 → 入库 → 检索全流程
- 下线 docker-compose 中的 `milvus`、`etcd` 服务
