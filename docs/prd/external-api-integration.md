# 外部系统集成技术方案：知识库文档上传与 RAG 检索

## 1. 概述

Yuxi-Know 完全支持通过外部系统或工具调用其 REST API，实现：
- **文档上传**：将文件或 URL 内容导入指定知识库
- **RAG 检索**：向知识库发起语义检索，获取结构化结果

所有接口均基于 HTTP REST + JWT Bearer Token 鉴权，无需改动任何代码即可直接对接。

---

## 2. 认证方式

所有知识库接口均需要管理员权限（`get_admin_user`），采用 OAuth2 Password Flow。

### 2.1 获取 Token

```http
POST /api/auth/token
Content-Type: application/x-www-form-urlencoded

username=admin&password=your_password
```

**响应示例：**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

后续所有请求在 Header 中携带：
```
Authorization: Bearer <access_token>
```

> Token 有效期到期后需重新获取。建议外部系统在调用失败（401）时自动刷新 Token。

---

## 3. 文档上传流程

文档从上传到可被检索分为三个阶段：**上传 → 解析 → 入库（Indexing）**。
可通过 `auto_index: true` 参数一次性完成全部流程。

### 3.1 前置：获取知识库 ID

```http
GET /api/knowledge/databases
Authorization: Bearer <token>
```

响应中每个库包含 `db_id` 字段，用于后续操作。

---

### 3.2 方式一：上传本地文件（推荐）

#### Step 1：上传文件到存储

```http
POST /api/knowledge/files/upload?db_id={db_id}
Authorization: Bearer <token>
Content-Type: multipart/form-data

file=@/path/to/document.pdf
```

**支持格式：** PDF、DOCX、TXT、MD、HTML、XLSX、CSV、PPTX 等（通过 `/api/knowledge/files/supported-types` 获取完整列表）。

**响应示例：**
```json
{
  "message": "File successfully uploaded",
  "file_path": "http://minio:9000/ref-{db_id}/document_1710000000000.pdf",
  "minio_path": "http://minio:9000/ref-{db_id}/document_1710000000000.pdf",
  "filename": "document.pdf",
  "content_hash": "sha256:abc123...",
  "same_name_files": [],
  "has_same_name": false
}
```

关键返回值：`file_path`（MinIO 存储路径），用于 Step 2。

---

#### Step 2：添加文档到知识库（含解析和自动入库）

```http
POST /api/knowledge/databases/{db_id}/documents
Authorization: Bearer <token>
Content-Type: application/json

{
  "items": ["http://minio:9000/ref-{db_id}/document_1710000000000.pdf"],
  "params": {
    "auto_index": true
  }
}
```

**params 可选参数：**

| 参数 | 类型 | 说明 |
|------|------|------|
| `auto_index` | bool | `true` 表示解析完成后自动入库，默认 `false` |
| `chunk_preset_id` | string | 切片预设 ID，控制分块策略 |
| `chunk_parser_config` | object | 自定义分块解析配置 |

此接口为异步任务，响应体包含 `task_id`，可通过以下接口轮询进度：

```http
GET /api/tasks/{task_id}
Authorization: Bearer <token>
```

任务状态字段 `status` 值：`pending` / `processing` / `done` / `failed`。

---

### 3.3 方式二：从 URL 抓取内容上传

```http
POST /api/knowledge/files/fetch-url
Authorization: Bearer <token>
Content-Type: application/json

{
  "url": "https://example.com/doc.pdf",
  "db_id": "{db_id}"
}
```

返回同 Step 1，拿到 `file_path` 后继续执行 Step 2。

---

### 3.4 方式三：手动分步控制（高级）

适合需要精确控制每个阶段的场景：

**解析阶段：**
```http
POST /api/knowledge/databases/{db_id}/documents/parse
Authorization: Bearer <token>
Content-Type: application/json

["file_id_1", "file_id_2"]
```

**入库阶段：**
```http
POST /api/knowledge/databases/{db_id}/documents/index
Authorization: Bearer <token>
Content-Type: application/json

{
  "file_ids": ["file_id_1", "file_id_2"],
  "params": {}
}
```

> 注意：Step 2（添加文档）返回的 `file_id` 才是解析/入库阶段需要的 ID，不是 MinIO 路径。

---

## 4. RAG 检索

### 4.1 标准检索

```http
POST /api/knowledge/databases/{db_id}/query
Authorization: Bearer <token>
Content-Type: application/json

{
  "query": "请问产品的退款政策是什么？",
  "meta": {
    "final_top_k": 5,
    "search_mode": "hybrid",
    "similarity_threshold": 0.2
  }
}
```

**meta 查询参数说明：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `final_top_k` | int | 10 | 返回的最终结果数量 |
| `similarity_threshold` | float | 0.2 | 相似度过滤阈值（0~1），低于该值的结果被过滤 |
| `search_mode` | string | `"vector"` | 检索模式：`vector`（向量）/ `keyword`（关键词）/ `hybrid`（混合） |
| `metric_type` | string | `"COSINE"` | 距离度量方式，`COSINE` 或 `L2` |
| `include_distances` | bool | `true` | 是否在结果中包含相似度分数 |
| `use_reranker` | bool | `false` | 是否使用重排序模型 |
| `recall_top_k` | int | 50 | 启用 Reranker 时的初始召回数量 |
| `file_name` | string | - | 按文件名过滤，支持模糊匹配 |

> `meta` 中的参数会临时覆盖知识库持久化的查询参数配置，仅对本次查询生效。

**响应示例：**
```json
{
  "status": "success",
  "result": [
    {
      "content": "退款申请需在购买后7天内提交...",
      "metadata": {
        "source": "退款政策.pdf",
        "chunk_id": "chunk_001",
        "file_id": "abc123",
        "chunk_index": 3
      },
      "distance": 0.92
    }
  ]
}
```

---

### 4.2 测试检索（不影响生产日志）

```http
POST /api/knowledge/databases/{db_id}/query-test
Authorization: Bearer <token>
Content-Type: application/json

{
  "query": "测试问题",
  "meta": {}
}
```

接口行为与 `/query` 一致，适合调试阶段使用。

---

## 5. 完整集成示例（Python）

```python
import httpx

BASE_URL = "http://localhost:5050"
DB_ID = "your-database-id"

def get_token(username: str, password: str) -> str:
    resp = httpx.post(f"{BASE_URL}/api/auth/token", data={
        "username": username,
        "password": password
    })
    resp.raise_for_status()
    return resp.json()["access_token"]

def upload_and_index(token: str, file_path: str, db_id: str) -> dict:
    headers = {"Authorization": f"Bearer {token}"}

    # Step 1: 上传文件
    with open(file_path, "rb") as f:
        resp = httpx.post(
            f"{BASE_URL}/api/knowledge/files/upload",
            params={"db_id": db_id},
            files={"file": f},
            headers=headers
        )
    resp.raise_for_status()
    minio_path = resp.json()["file_path"]

    # Step 2: 添加到知识库并自动入库
    resp = httpx.post(
        f"{BASE_URL}/api/knowledge/databases/{db_id}/documents",
        json={"items": [minio_path], "params": {"auto_index": True}},
        headers=headers
    )
    resp.raise_for_status()
    return resp.json()

def rag_query(token: str, db_id: str, question: str, top_k: int = 5) -> list[dict]:
    headers = {"Authorization": f"Bearer {token}"}
    resp = httpx.post(
        f"{BASE_URL}/api/knowledge/databases/{db_id}/query",
        json={
            "query": question,
            "meta": {
                "final_top_k": top_k,
                "search_mode": "hybrid",
                "similarity_threshold": 0.2
            }
        },
        headers=headers
    )
    resp.raise_for_status()
    return resp.json().get("result", [])

# 使用示例
token = get_token("admin", "your_password")
upload_and_index(token, "/path/to/doc.pdf", DB_ID)
results = rag_query(token, DB_ID, "产品退款政策是什么？")
for chunk in results:
    print(chunk["content"])
    print(f"来源: {chunk['metadata']['source']}, 相似度: {chunk.get('distance', 'N/A')}")
```

---

## 6. 关键接口速查表

| 用途 | Method | 路径 |
|------|--------|------|
| 获取 Token | POST | `/api/auth/token` |
| 获取知识库列表 | GET | `/api/knowledge/databases` |
| 上传文件 | POST | `/api/knowledge/files/upload?db_id={db_id}` |
| 从 URL 抓取 | POST | `/api/knowledge/files/fetch-url` |
| 添加文档（含解析/入库） | POST | `/api/knowledge/databases/{db_id}/documents` |
| 手动触发解析 | POST | `/api/knowledge/databases/{db_id}/documents/parse` |
| 手动触发入库 | POST | `/api/knowledge/databases/{db_id}/documents/index` |
| RAG 检索 | POST | `/api/knowledge/databases/{db_id}/query` |
| 检索测试 | POST | `/api/knowledge/databases/{db_id}/query-test` |
| 查询任务状态 | GET | `/api/tasks/{task_id}` |
| 获取查询参数配置 | GET | `/api/knowledge/databases/{db_id}/query-params` |

---

## 7. 注意事项

1. **文件去重**：相同内容（SHA256）的文件在同一知识库中只能上传一次，重复上传返回 `409`。
2. **异步任务**：文档添加（解析 + 入库）为异步任务，需轮询 `/api/tasks/{task_id}` 确认完成后再检索，否则新文档不会出现在检索结果中。
3. **知识库类型**：不同类型的知识库（Milvus、LightRAG 等）支持的 `meta` 参数不同。可通过 `GET /api/knowledge/databases/{db_id}/query-params` 获取当前库支持的参数列表。
4. **Dify 类型知识库**：通过 Dify 接入的知识库不支持文档上传/解析/入库操作，只支持查询。
5. **超时配置**：文档解析（尤其是 PDF OCR）可能耗时较长，建议外部系统将 HTTP 超时设置为 300 秒以上，或采用异步轮询模式。
