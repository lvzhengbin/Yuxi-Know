# BM25 全文检索接入方案

> 文档创建日期：2026-03-18
> 相关文件：`src/knowledge/implementations/milvus.py`

---

## 1. BM25 是什么？

**BM25（Best Matching 25）** 是全文搜索领域最经典的词频统计排名算法，Elasticsearch / Lucene 的默认相关性算法就是它。

### 核心公式

```
BM25(q, d) = Σ IDF(qi) × TF(qi, d) × (k1+1) / (TF(qi,d) + k1×(1-b+b×|d|/avgdl))
```

三个核心因子：
- **TF（词频）**：词在文档中出现越多，分数越高，但有饱和限制（非线性增长，避免高频堆砌）
- **IDF（逆文档频率）**：在整个语料中越罕见的词权重越高，"的/了/是"等停用词权重接近 0
- **文档长度归一化**：短文档中出现 1 次比长文档中出现 1 次更有意义

### 与当前 LIKE 扫描的对比

| | Milvus LIKE 扫描（当前） | BM25（目标） |
|--|--|--|
| 实现原理 | `content like "%kw%"` 全量遍历 | 预建倒排索引，只访问含该词的文档 |
| 时间复杂度 | O(N)，数据越多越慢 | O(log N) 左右 |
| 评分方式 | 简单词频统计 | TF-IDF 加权 + 文档长度归一化 |
| 多词查询 | 只看是否包含关键词 | 综合考虑每个词的重要性与稀有度 |
| 停用词处理 | 无 | IDF 自动降权 |

### 与向量语义检索的互补关系

| 场景 | 向量检索 | BM25 |
|------|----------|------|
| 语义相近但用词不同 | ✅ 强 | ❌ 弱 |
| 专有名词、产品型号 | ❌ 弱 | ✅ 强 |
| 缩写、代码片段 | ❌ 弱 | ✅ 强 |
| 口语化/模糊查询 | ✅ 强 | ❌ 弱 |

**结论**：混合检索 = 向量语义 + BM25，两者互补，效果优于任何单一方式。

---

## 2. 技术选型

项目使用 **Milvus v2.5.6 + pymilvus >= 2.5.8**，该版本已**内置 BM25 全文搜索**，无需引入 Elasticsearch 等外部服务。

实现原理：在 Schema 中新增稀疏向量字段，声明 BM25 Function，Milvus 在文档插入时自动将 `content` 文本转换为 BM25 稀疏向量，检索时传入原始文本即可。

---

## 3. 实现方案

改动范围：仅 `src/knowledge/implementations/milvus.py`，三处修改。

### 3.1 Schema 变更（`_create_new_collection`）

在 `content` 字段上开启分词器，新增 BM25 稀疏向量字段，声明 BM25 Function：

```python
from pymilvus import Function, FunctionType

fields = [
    FieldSchema(name="id", dtype=DataType.VARCHAR, max_length=100, is_primary=True),
    FieldSchema(
        name="content",
        dtype=DataType.VARCHAR,
        max_length=65535,
        enable_analyzer=True,                   # 开启分词
        analyzer_params={"type": "chinese"},    # 中文分词（Milvus 2.5 内置）
    ),
    FieldSchema(name="source", dtype=DataType.VARCHAR, max_length=500),
    FieldSchema(name="chunk_id", dtype=DataType.VARCHAR, max_length=100),
    FieldSchema(name="file_id", dtype=DataType.VARCHAR, max_length=100),
    FieldSchema(name="chunk_index", dtype=DataType.INT64),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=embedding_dim),
    FieldSchema(name="content_sparse", dtype=DataType.SPARSE_FLOAT_VECTOR),  # 新增 BM25 稀疏向量
]

schema = CollectionSchema(fields=fields, description=f"...")

# 声明 BM25 函数：插入时自动将 content 转为稀疏向量
bm25_func = Function(
    name="bm25",
    function_type=FunctionType.BM25,
    input_field_names=["content"],
    output_field_names=["content_sparse"],
)
schema.add_function(bm25_func)

collection = Collection(name=collection_name, schema=schema, ...)

# 向量索引（原有）
collection.create_index("embedding", {
    "metric_type": "COSINE",
    "index_type": "IVF_FLAT",
    "params": {"nlist": 1024},
})

# BM25 稀疏索引（新增）
collection.create_index("content_sparse", {
    "index_type": "SPARSE_INVERTED_INDEX",
    "metric_type": "BM25",
})
```

### 3.2 纯 BM25 检索（`keyword` 模式）

替换原来的 `content like "%kw%"` 全扫描，传入原始文本，Milvus 内部自动分词并用倒排索引检索：

```python
from pymilvus import AnnSearchRequest

sparse_req = AnnSearchRequest(
    data=[query_text],              # 传原始文本，无需手动分词
    anns_field="content_sparse",
    param={"metric_type": "BM25"},
    limit=keyword_top_k,
    expr=file_expr,
)
results = collection.search(
    data=[query_text],
    anns_field="content_sparse",
    param={"metric_type": "BM25"},
    limit=keyword_top_k,
    expr=file_expr,
    output_fields=["content", "source", "chunk_id", "file_id", "chunk_index"],
)
```

### 3.3 混合检索用 RRF 融合（`hybrid` 模式）

用 Milvus 原生 `hybrid_search` + `RRFRanker` 替代当前的手动 max 融合：

```python
from pymilvus import AnnSearchRequest, RRFRanker

dense_req = AnnSearchRequest(
    data=query_embedding,
    anns_field="embedding",
    param={"metric_type": "COSINE", "params": {"nprobe": 10}},
    limit=recall_top_k,
)
sparse_req = AnnSearchRequest(
    data=[query_text],
    anns_field="content_sparse",
    param={"metric_type": "BM25"},
    limit=recall_top_k,
)

results = collection.hybrid_search(
    reqs=[dense_req, sparse_req],
    rerank=RRFRanker(k=60),     # k=60 是业界经验最优值
    limit=final_top_k,
    output_fields=["content", "source", "chunk_id", "file_id", "chunk_index"],
)
```

#### 为什么用 RRF 而不是 max？

RRF（Reciprocal Rank Fusion）公式：

```
RRF_score(chunk) = Σ 1 / (k + rank_i)
```

只依赖**排名**，不依赖原始分数值，因此不受两路分数量纲不一致的影响（向量余弦相似度 vs BM25 分数完全不可比）。这是目前混合检索融合的业界最佳实践。

---

## 4. 注意事项

### 现有集合需要重建

`enable_analyzer` 是 Schema 级别的变更，已存在的 Milvus 集合**无法热更新**，需要：
1. 删除现有集合
2. 用新 Schema 重建集合
3. 重新插入所有文档（重新索引）

已有知识库的文档需要触发"重新索引"操作。代码中已有 `delete_file_chunks_only` + `index_file` 的重索引流程可复用。

### 中文分词

Milvus 2.5 内置的 `chinese` analyzer 使用 ICU 分词，能正确处理中文语境下的词边界。对于纯英文或混合文档，可使用 `standard` analyzer。

### 插入时无需额外操作

BM25 Function 声明后，插入文档时只需提供 `content` 字段，`content_sparse` 由 Milvus 自动填充，插入代码无需改动。

---

## 5. 改动总结

| 位置 | 改动类型 | 说明 |
|------|----------|------|
| `_create_new_collection` | Schema 变更 | content 字段开启 analyzer，新增 content_sparse 字段、BM25 Function、稀疏索引 |
| `aquery` keyword 分支 | 替换实现 | 用 BM25 AnnSearchRequest 替代 LIKE 扫描 |
| `aquery` hybrid 分支 | 替换实现 | 用 `hybrid_search` + `RRFRanker` 替代手动 max 融合 |
| `index_file` / `insert` | **无需改动** | BM25 稀疏向量由 Milvus 自动生成 |
