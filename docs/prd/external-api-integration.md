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
    "content_hashes": {
      "http://minio:9000/ref-{db_id}/document_1710000000000.pdf": "sha256:abc123..."
    },
    "auto_index": true
  }
}
```

> `content_hashes` 的键为 Step 1 返回的 `file_path`，值为 Step 1 返回的 `content_hash`。**当 items 为 MinIO 文件 URL 时为必填**，缺少此参数会导致添加失败（`Missing content_hash for file`）。

**params 参数：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `content_hashes` | object | **是**（MinIO 文件时） | MinIO URL → content_hash 的映射，值来自 Step 1 响应的 `content_hash` 字段 |
| `parent_id` | string | 否 | 目标文件夹的 `file_id`，不传则上传到知识库根目录。见 [3.5 上传到指定目录](#35-上传到指定目录文件夹) |
| `auto_index` | bool | 否 | `true` 表示解析完成后自动入库，默认 `false` |
| `chunk_preset_id` | string | 否 | 切片预设 ID，控制分块策略 |
| `chunk_parser_config` | object | 否 | 自定义分块解析配置 |

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

### 3.4 上传到指定目录（文件夹）

默认情况下，文档会上传到知识库根目录。通过在 Step 2 的 `params` 中传入 `parent_id`（目标文件夹的 `file_id`），可以将文档放入指定子目录。

#### 获取现有文件夹 ID

```http
GET /api/knowledge/databases/{db_id}
Authorization: Bearer <token>
```

响应中的 `files` 字段包含该知识库所有文件和文件夹，过滤 `is_folder: true` 的条目即可获取文件夹列表：

```json
{
  "files": {
    "folder_abc123": {
      "file_id": "folder_abc123",
      "filename": "产品文档",
      "is_folder": true,
      "parent_id": null
    },
    "folder_def456": {
      "file_id": "folder_def456",
      "filename": "技术规范",
      "is_folder": true,
      "parent_id": "folder_abc123"
    }
  }
}
```

#### 创建新文件夹（可选）

如果目标目录不存在，可先创建：

```http
POST /api/knowledge/databases/{db_id}/folders
Authorization: Bearer <token>
Content-Type: application/json

{
  "folder_name": "新目录名称",
  "parent_id": null
}
```

> `parent_id` 为 `null` 表示在根目录下创建；传入某个文件夹的 `file_id` 则在该文件夹下创建子目录。

响应中包含新建文件夹的 `file_id`，用于后续上传。

#### 上传到指定目录

在 Step 2 的 `params` 中加入 `parent_id`：

```http
POST /api/knowledge/databases/{db_id}/documents
Authorization: Bearer <token>
Content-Type: application/json

{
  "items": ["http://minio:9000/ref-{db_id}/document_1710000000000.pdf"],
  "params": {
    "content_hashes": {
      "http://minio:9000/ref-{db_id}/document_1710000000000.pdf": "sha256:abc123..."
    },
    "parent_id": "folder_abc123",
    "auto_index": true
  }
}
```

---

### 3.5 方式三：手动分步控制（高级）

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

def upload_and_index(token: str, file_path: str, db_id: str, parent_id: str | None = None) -> dict:
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
    upload_result = resp.json()
    minio_path = upload_result["file_path"]
    content_hash = upload_result["content_hash"]

    # Step 2: 添加到知识库并自动入库
    # content_hashes 为必填：key 为 MinIO URL，value 为 content_hash
    # parent_id 可选：指定目标文件夹，不传则上传到根目录
    params = {
        "content_hashes": {minio_path: content_hash},
        "auto_index": True,
    }
    if parent_id:
        params["parent_id"] = parent_id

    resp = httpx.post(
        f"{BASE_URL}/api/knowledge/databases/{db_id}/documents",
        json={"items": [minio_path], "params": params},
        headers=headers
    )
    resp.raise_for_status()
    return resp.json()


def get_folder_id(token: str, db_id: str, folder_name: str) -> str | None:
    """按名称查找文件夹 ID，不存在返回 None"""
    headers = {"Authorization": f"Bearer {token}"}
    resp = httpx.get(f"{BASE_URL}/api/knowledge/databases/{db_id}", headers=headers)
    resp.raise_for_status()
    files = resp.json().get("files", {})
    for file_id, info in files.items():
        if info.get("is_folder") and info.get("filename") == folder_name:
            return file_id
    return None


def create_folder(token: str, db_id: str, folder_name: str, parent_id: str | None = None) -> str:
    """创建文件夹并返回其 file_id"""
    headers = {"Authorization": f"Bearer {token}"}
    resp = httpx.post(
        f"{BASE_URL}/api/knowledge/databases/{db_id}/folders",
        json={"folder_name": folder_name, "parent_id": parent_id},
        headers=headers
    )
    resp.raise_for_status()
    return resp.json()["file_id"]

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

# 上传到根目录
upload_and_index(token, "/path/to/doc.pdf", DB_ID)

# 上传到指定目录（先查找，不存在则创建）
folder_id = get_folder_id(token, DB_ID, "产品文档")
if folder_id is None:
    folder_id = create_folder(token, DB_ID, "产品文档")
upload_and_index(token, "/path/to/doc.pdf", DB_ID, parent_id=folder_id)

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
| 获取知识库详情（含文件夹列表） | GET | `/api/knowledge/databases/{db_id}` |
| 上传文件 | POST | `/api/knowledge/files/upload?db_id={db_id}` |
| 从 URL 抓取 | POST | `/api/knowledge/files/fetch-url` |
| 创建文件夹 | POST | `/api/knowledge/databases/{db_id}/folders` |
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
