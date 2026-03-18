# Yuxi-Know RAG 检索模块深度分析

> 文档创建日期：2026-03-18
> 分析范围：知识库检索模块（Milvus + LightRAG 双路实现）

---

## 1. 系统架构概览

Yuxi-Know 的 RAG 检索模块采用**双路知识库后端**设计，支持两种核心实现：

| 实现 | 存储组件 | 适用场景 |
|------|----------|----------|
| **MilvusKB** | Milvus 向量数据库 | 纯向量/关键词/混合检索，适合通用文档 |
| **LightRagKB** | Milvus + Neo4J + JSON | 图增强 RAG，适合需要实体关系推理的场景 |
| **DifyKB** | Dify 外部服务 | 对接 Dify 平台 |

整体数据流：

```
原始文件（PDF/Word/Excel/...）
    ↓ 文件解析（Docling / LangChain Loaders）
Markdown 文本（存储于 MinIO）
    ↓ 智能分块（ragflow_like 引擎）
文本 Chunks
    ↓ 向量化（Embedding 模型，批大小=40）
(Chunk, Vector) 对
    ↓ 存储
    ├── MilvusKB → Milvus（单集合）
    └── LightRagKB → Milvus（3个集合）+ Neo4J（实体关系图）

查询时：
用户 Query
    ↓ 向量化
    ↓ 向量搜索 / 关键词搜索 / 混合搜索
初步召回（recall_top_k，默认50）
    ↓ 相似度阈值过滤（默认0.2）
    ↓ [可选] Reranker 精排
最终结果（final_top_k，默认10）
```

核心文件路径：

```
src/knowledge/
├── base.py                             # KnowledgeBase 抽象基类
├── manager.py                          # 多类型知识库统一管理
├── factory.py                          # 工厂方法
├── indexing.py                         # 文件→Markdown 转换
├── implementations/
│   ├── milvus.py                       # MilvusKB 实现
│   └── lightrag.py                     # LightRagKB 实现
└── chunking/ragflow_like/
    ├── presets.py                      # 分块预设与参数解析
    ├── dispatcher.py                   # 分块路由
    ├── nlp.py                          # NLP 工具（token计数、标题识别、合并算法）
    └── parsers/
        ├── general.py                  # 通用分块
        ├── qa.py                       # 问答文档分块
        ├── book.py                     # 书籍/手册分块
        └── laws.py                     # 法规分块

src/models/
├── embed.py                            # Embedding 模型管理
└── rerank.py                           # Reranker 模型管理
```

---

## 2. 文档处理流程

### 2.1 文件解析（indexing.py）

文件首先被转换为统一的 Markdown 格式，存入 MinIO 对象存储：

- **Office 文档**（docx/xlsx/pptx）：使用 Docling 进行结构化解析，保留表格、标题层级
- **PDF**：LangChain `PyPDFLoader` 或 Docling
- **HTML/JSON/TXT**：对应 LangChain Loaders
- **图片**：上传到 MinIO，以 Markdown 图片 URL 形式嵌入

这种统一为 Markdown 的做法简化了后续分块逻辑——所有类型文档共用同一套分块引擎。

### 2.2 分块参数解析（presets.py: resolve_chunk_processing_params）

分块参数遵循**四级优先级覆盖**机制：

```
预设默认值（preset_defaults）
    ← 知识库级配置（kb_additional_params.chunk_parser_config）
        ← 文件级配置（file_processing_params.chunk_parser_config）
            ← 请求级配置（request_params.chunk_parser_config）
```

优先级从低到高，使用 `deep_merge` 深度合并，支持精细控制。

---

## 3. 分块策略详解

### 3.1 四种预设

| 预设 ID | 适用场景 | 核心参数 | RAPTOR | GraphRAG |
|---------|----------|----------|--------|----------|
| `general` | 通用文档 | chunk_token_num=512, delimiter="\n" | ✅ (max_token=256) | ✅ (light) |
| `qa` | FAQ/题库/问答手册 | 优先抽取 Q-A 对结构 | ❌ | ❌ |
| `book` | 教材/手册/长章节 | 强化章节标题识别，层级合并 | ❌ | ❌ |
| `laws` | 法律法规/制度规范 | 按法条层级组织合并 | ❌ | ❌ |

### 3.2 通用分块算法（naive_merge）

核心实现在 `nlp.py: naive_merge`：

```python
def naive_merge(sections, chunk_token_num=128, delimiter="\n。；！？", overlapped_percent=0):
    # 按分隔符切分 sections
    # 按 token 数量滚动合并：
    #   当前 chunk 超过 threshold = chunk_token_num * (1 - overlap/100) 时，
    #   创建新 chunk，并从上一 chunk 末尾截取 overlap 部分作为前缀
```

**重叠机制**：`overlapped_percent` 控制相邻 chunk 的文本重叠量，默认为 0（无重叠）。当前通用预设未启用重叠，会造成语义跨 chunk 断裂。

**Token 计数**：采用轻量近似计数（`count_tokens`），匹配英文单词+数字+CJK 单字，**不使用 tokenizer**，避免引入依赖，但对子词（subword）型语言精度有限。

### 3.3 标题识别（bullets_category + is_probable_heading_line）

通过正则模式组识别文档中的标题层级：

- 组0：中文法规章节（"第X编/章/节/条"）
- 组1：数字编号（"1.2.3"）
- 组2：中文列表（"一、二、三"，"（一）"）
- 组3：英文标题（"Chapter", "Section", "Article"）
- 组4：Markdown 标题（`#`到`######`，H1/H2 权重提升至4倍/3倍）

标题候选还需通过 `is_probable_heading_line` 验证：
- 长度 ≤ 96 字符 且 token 数 ≤ 72
- 前24字符内不含 `，。；！？:：` 等句末/句中标点
- 不含 HTML 表格残留标签

### 3.4 结构化合并（tree_merge / hierarchical_merge）

`book`/`laws` 预设使用树形合并：将识别出的标题层级构建树结构，子节点内容归属至父标题下，保持语义完整性，避免章节内容被随机切断。

### 3.5 RAPTOR 支持

`general` 预设默认启用 RAPTOR（递归摘要树），配置如下：
```python
"raptor": {
    "use_raptor": True,
    "max_token": 256,          # 摘要块最大 token
    "threshold": 0.1,          # 聚类阈值
    "max_cluster": 64,         # 最大聚类数
    "random_seed": 0,
}
```
RAPTOR 对大规模文档构建多层次摘要索引，提升高层次问题（全局摘要性问题）的召回率。

---

## 4. 向量化实现

### 4.1 模型管理（embed.py）

```python
class BaseEmbeddingModel(ABC):
    batch_size = 40           # 默认批大小，兼顾吞吐与内存

    async def abatch_encode(self, messages, batch_size=None):
        # 分批异步请求 Embedding API
        # 返回 [[float,...], ...]
```

支持的提供者：
- **Ollama**（本地部署）
- **OpenAI 兼容 API**（SiliconFlow / vLLM / 自定义）
- 通过 `select_embedding_model(model_id)` 统一获取

### 4.2 向量化流程

1. 读取 MinIO 中的 Markdown 内容
2. 分块 → `chunks` 列表
3. 批量提取 `chunk["content"]` 文本
4. 调用 `embedding_function(texts)` 异步批处理
5. 将向量与 chunk 元数据一起插入 Milvus

---

## 5. 向量存储（Milvus）

### 5.1 集合 Schema

```
字段            类型            说明
id              VARCHAR(100)    主键，格式：{file_id}_chunk_{idx}
content         VARCHAR(65535)  chunk 文本内容
source          VARCHAR(500)    来源文件名
chunk_id        VARCHAR(100)    与 id 相同
file_id         VARCHAR(100)    所属文件 ID
chunk_index     INT64           chunk 在文件中的序号
embedding       FLOAT_VECTOR    向量（维度由 Embedding 模型决定，默认1024）
```

### 5.2 索引配置

```python
index_params = {
    "metric_type": "COSINE",    # 余弦相似度
    "index_type": "IVF_FLAT",   # 倒排平坦索引
    "params": {"nlist": 1024}   # 聚类桶数
}
```

**IVF_FLAT** 索引的特性：
- 搜索参数 `nprobe=10`（默认），即只扫描10个最近邻桶
- 精度与速度的权衡点；nlist=1024 适合中等规模数据集（百万级）

### 5.3 LightRAG 的三路存储

LightRAG 为每个知识库创建三个 Milvus 集合：
- `{db_id}_chunks`：原始文本块及其向量
- `{db_id}_entities`：LLM 识别出的实体（人物/地点/组织/事件等）及其向量
- `{db_id}_relationships`：实体间关系向量

同时在 Neo4J 中存储完整的知识图谱结构（节点+关系）。

---

## 6. 检索策略

### 6.1 MilvusKB 三种检索模式

#### 向量检索（vector）
```python
search_params = {"metric_type": "COSINE", "params": {"nprobe": 10}}
results = collection.search(
    data=query_embedding,
    anns_field="embedding",
    param=search_params,
    limit=recall_top_k,       # 启用 reranker 时默认50，否则=final_top_k
    expr=file_expr,           # 可按文件名过滤
)
# 过滤：similarity < similarity_threshold（默认0.2）则丢弃
```

#### 关键词检索（keyword）
```python
# 将 query 按空格/逗号/分号分词
keywords = re.split(r"[\s,，;；]+", query_text)

# 构建 OR 表达式（全文 LIKE 匹配）
expr = 'content like "%kw1%" or content like "%kw2%"'

# 评分：关键词出现频次归一化
score = match_count / max_match_count_in_results
```

**局限**：使用 Milvus 的 `content like "%kw%"` 实现，本质是全扫描，无倒排索引，大数据量下性能较差。

#### 混合检索（hybrid）
向量结果与关键词结果取**并集**，按 `chunk_id` 去重，取 `max(vector_score, keyword_score)` 作为最终分数，再降序排列。

**问题**：混合策略过于简单，两种分数量纲不一致（余弦相似度 vs 归一化词频），直接取 max 可能导致语义相关度高但关键词不匹配的文档被低分关键词结果"压制"。

### 6.2 LightRAG 五种检索模式

通过 `QueryParam.mode` 控制：

| 模式 | 说明 |
|------|------|
| `naive` | 纯向量检索，与 Milvus 模式类似 |
| `local` | 局部图检索：基于实体相关的局部子图 |
| `global` | 全局图检索：基于高连通性实体的全局视角 |
| `hybrid` | local + global 混合 |
| `mix` | 向量检索 + 图检索融合（**默认**，效果最佳） |

`mix` 模式同时返回 chunks、entities、relationships，上层可根据需要选择性使用。

---

## 7. 重排序机制

### 7.1 架构设计

```python
class BaseReranker(ABC):
    timeout = 30s               # HTTP 请求超时

    async def acompute_score(sentence_pairs, batch_size=32, normalize=True):
        # sentence_pairs = [query, [doc1, doc2, ...]]
        # 分批（默认32个文档/批）发送到 Reranker API
        # normalize=True 时对原始分数应用 sigmoid 归一化到 (0,1)
        # 批次失败时填充 0.5 作为默认分数（保守策略）
```

支持的提供者：
- **OpenAIReranker**：SiliconFlow / vLLM 等 OpenAI 兼容 Reranker API
- **DashscopeReranker**：阿里云 DashScope（ text-rerank 系列）

### 7.2 召回-重排序管道

```
query → 向量/关键词/混合召回（recall_top_k，默认50）
      → 相似度阈值过滤
      → Reranker 精排（对所有候选文档打分）
      → 按 rerank_score 重新排序
      → 截取 final_top_k（默认10）
```

**参数配置**：
- `recall_top_k`（默认50）：初步召回数量，在 reranker 开启时有效
- `final_top_k`（默认10）：最终返回数量
- Reranker `max_length`（默认512）：每个文档的最大 token 长度

### 7.3 当前问题

- 批次失败的默认分数是 `0.5`（sigmoid(0)），与正常文档分数混在一起，可能影响排序
- 每次查询都新建并关闭 `aiohttp.ClientSession`，频繁的连接创建有性能开销
- Reranker 为**可选组件**，默认关闭，用户需要手动配置

---

## 8. 上下文构建

检索结果直接以结构化格式返回给上层（Agent/对话模块），由上层决定如何组织 Prompt：

**MilvusKB 返回格式**：
```python
{
    "content": "文档片段文本",
    "metadata": {
        "source": "filename.pdf",
        "chunk_id": "abc123_chunk_0",
        "file_id": "abc123",
        "chunk_index": 0
    },
    "score": 0.87,         # 向量相似度或关键词频率分数
    "rerank_score": 0.93,  # Reranker 精排分数（如果启用）
    "distance": 0.87       # 原始距离值
}
```

**LightRAG 返回格式**（`only_need_context=True`）：
```python
{
    "chunks": [{"content": "...", "source": "...", "score": 0.9}],
    "entities": [{"name": "实体名", "description": "..."}],
    "relationships": [{"source": "A", "target": "B", "relation": "..."}],
    "references": [...]
}
```

---

## 9. 当前不足与待优化点

### 9.1 分块层面

**问题1：重叠为零**
`general` 预设的 `overlapped_percent` 默认为 0，语义连续的内容被切断后两个相邻 chunk 没有任何过渡文本，跨 chunk 的查询召回率低。

**问题2：Token 计数不精确**
`count_tokens` 基于正则近似（英文单词 + CJK 单字），实际 BPE/SentencePiece tokenizer 的 token 数可能差异较大，可能导致 chunk 实际超出模型上下文限制。

**问题3：无语义感知分块**
所有分块均基于字符级规则（分隔符、字数限制），没有感知段落语义边界。例如，一张完整的表格可能被切断在两个 chunk 中；一段完整的论述因超过 512 token 而被强行截断。

**问题4：元数据未编码进 Chunk**
chunk 内容仅为纯文本，缺少结构化上下文（所属章节标题、文档类型、页码等）。查询时无法利用这些信息做精细过滤。

**问题5：分块预设较少**
目前只有 4 种预设，缺乏：代码文档分块（保留代码块完整性）、表格专项分块、多语言混合文档分块等。

---

### 9.2 向量化层面

**问题6：单一 Embedding 模型**
每个知识库绑定单一 Embedding 模型，无法针对不同类型内容（代码 vs 自然语言 vs 表格数值）选用更合适的模型。

**问题7：查询-文档不对称**
`query_text` 和 chunk 内容直接使用同一 embedding 函数编码，未使用针对不对称（asymmetric）检索优化的 embedding（如 `e5-mistral-instruct` 的 `query:` 前缀指令调优）。

---

### 9.3 检索策略层面

**问题8：混合检索分数融合粗糙**
当前混合模式的分数融合仅取 `max(vector_score, keyword_score)`，两种分数量纲完全不同，直接比较无意义。正确的做法应使用 RRF（Reciprocal Rank Fusion）或学习型融合权重。

**问题9：关键词检索效率低**
使用 Milvus `content like "%keyword%"` 实现关键词检索，无法利用倒排索引，本质上是全量扫描，大数据量下延迟显著。

**问题10：无查询扩展/改写**
用户的原始 query 直接用于检索，没有：
- 查询扩展（同义词、相关词）
- HyDE（假设文档嵌入）：先让 LLM 生成一段假设答案，再用假设答案向量检索
- 多路查询（生成多个角度的 query 分别检索，结果取并集）

**问题11：相似度阈值缺乏自适应**
`similarity_threshold` 是固定配置值，不同 Embedding 模型的分数分布差异很大，同样的 0.2 阈值在不同模型下效果完全不同。

**问题12：无负反馈/上下文感知**
检索不感知对话历史，相同问题在多轮对话中每次独立检索，无法利用之前的相关 chunk 作为先验。

---

### 9.4 重排序层面

**问题13：Reranker 默认关闭**
精排对召回精度提升显著，但当前作为可选配置，用户体验上与不使用 reranker 无明显引导。

**问题14：Reranker max_length 偏短**
默认 `max_length=512`，如果 chunk 本身超过 512 token，文档末尾部分被截断，可能丢失关键信息。

**问题15：无交叉编码器（Cross-Encoder）本地支持**
目前只支持远程 API 调用 Reranker，无法使用本地部署的 Cross-Encoder 模型（如 BGE-Reranker），增加了延迟和对外部服务的依赖。

---

### 9.5 LightRAG 集成层面

**问题16：实体提取依赖 LLM**
LightRAG 在索引时调用 LLM 做实体识别和关系提取，成本高且速度慢，不适合大批量文档快速入库。

**问题17：图谱结果利用不充分**
`mix` 模式下返回了 entities 和 relationships，但上层 Agent 模块对这些结构化信息的利用较少，大多只是将 chunks 拼接成上下文。

---

## 10. 提升检索召回精确度的技术思考

### 10.1 短期优化（较低成本，效果明显）

**① 开启 Chunk 重叠**
将 `general` 预设的 `overlapped_percent` 从 0 调整为 10~20%（约50~100 token 的重叠）。语义跨 chunk 的内容能在两个 chunk 中都被检索到，显著提升召回率。对存储量影响约10~20%。

**② 调优相似度阈值**
为每种 Embedding 模型设定经过统计的合理阈值（而非统一的 0.2），或基于用户反馈动态调整。

**③ 默认启用 Reranker**
将 Reranker 作为推荐选项展示给用户，提供开箱即用的 BGE-Reranker 配置示例。

**④ 提升关键词检索能力**
引入 Elasticsearch / Meilisearch 作为关键词索引后端，替换当前的 Milvus LIKE 扫描，获得真正的 BM25 全文检索能力。

### 10.2 中期优化（中等成本，收益显著）

**⑤ 实现 RRF 混合融合**
Reciprocal Rank Fusion 公式：
```
RRF_score(chunk) = Σ 1 / (k + rank_i)    k=60（经验值）
```
不依赖分数量纲，只依赖排名，是目前业界混合检索融合的最佳实践。

**⑥ 引入 HyDE（假设文档嵌入）**
```
query → LLM(生成假设回答) → embedding(假设回答) → 向量检索
```
假设回答与文档内容在语义空间更接近，能提升对"答案型查询"的召回率。适合知识密集型问答场景。

**⑦ 父子 Chunk 策略**
索引时将文档按两个粒度切分：
- **小 Chunk**（~128 token）：用于精确向量匹配
- **大 Chunk**（~512 token，父节点）：作为返回给 LLM 的上下文

检索时用小 chunk 找到匹配点，返回其父 chunk 给 LLM，兼顾检索精度和上下文完整性。

**⑧ 语义分块（Semantic Chunking）**
使用 Embedding 模型计算相邻句子的语义相似度，在语义跳变处切分，而非固定字数切分。可以有效避免一段完整论述被割裂。

### 10.3 长期优化（较高成本，战略价值高）

**⑨ 查询改写（Query Rewriting）**
```
原始 query → LLM(生成多个改写版本) → 多路并行检索 → 结果去重合并
```
多角度查询能覆盖原始表达可能遗漏的相关内容，尤其适合模糊/口语化的用户输入。

**⑩ 对话历史感知检索（Contextual Retrieval）**
引入对话历史，在检索前将上下文注入 query：
```python
contextualized_query = f"[对话历史]\n{history}\n\n[当前问题]\n{query}"
```

**⑪ 多向量 / 晚期交互（Late Interaction）**
引入 ColBERT 风格的晚期交互模型，不将文档压缩到单个向量，而是保留每个 token 的向量，查询时进行 MaxSim 操作。可在不增加存储太多的情况下大幅提升精度。

**⑫ 自适应 RAG**
结合意图分类决定检索策略：
- 事实性问题 → 精确向量检索 + Reranker
- 综合推理问题 → LightRAG mix 模式（利用知识图谱）
- 统计/计算问题 → 直接调用工具，不检索

**⑬ Chunk 元数据增强（Contextual Embedding）**
参考 Anthropic Contextual Retrieval 方案：在 embedding 前，为每个 chunk 生成一段描述其所在上下文的摘要句，将摘要句前置到 chunk 内容中，使 embedding 包含更丰富的上下文语义。

---

## 11. 关键参数速查

| 参数 | 默认值 | 建议调整 | 说明 |
|------|--------|----------|------|
| `chunk_token_num` | 512 | 256~1024，按场景 | 每个 chunk 的目标 token 数 |
| `overlapped_percent` | 0 | 10~20 | chunk 重叠百分比 |
| `recall_top_k` | 50 | 50~100 | 初步召回候选数（开启 reranker 时） |
| `final_top_k` | 10 | 5~20 | 最终返回给 LLM 的 chunk 数 |
| `similarity_threshold` | 0.2 | 按模型校准 | 相似度过滤阈值 |
| `nprobe` | 10 | 10~50 | Milvus IVF 搜索桶数，越高精度越好但越慢 |
| `nlist` | 1024 | 与数据量挂钩 | IVF 聚类桶数，建议 = sqrt(数据量) |
| Reranker `batch_size` | 32 | 16~32 | 重排序批大小 |
| Reranker `max_length` | 512 | 512~2048 | 文档最大 token 长度（截断限制） |
| Embedding `batch_size` | 40 | 20~100 | 向量化批大小 |

---

## 12. 总结

Yuxi-Know 的 RAG 检索模块已具备良好的基础架构：
- **双路知识库后端**（Milvus 纯向量 + LightRAG 图增强）满足不同场景需求
- **灵活的四级参数覆盖**机制支持细粒度配置
- **三种检索模式**（向量/关键词/混合）和**可选 Reranker 精排**形成了基本的检索管道
- **LightRAG** 的 `mix` 模式提供了业界领先的图增强 RAG 能力

主要短板集中在：**分块质量**（无重叠、无语义感知）、**混合检索融合方式粗糙**、**关键词检索底层效率低**、以及**缺乏查询扩展**等优化手段。

优先级最高的优化项建议：
1. 开启 chunk 重叠（最小改动，收益明显）
2. 实现 RRF 混合融合替代当前的 max 策略
3. 引入 BM25 全文检索替代 LIKE 扫描
4. 默认推荐 Reranker 使用
