# Yuxi-Know Skills 设计方案：yuxi-upload & yuxi-rag

## 1. 概述

将 Yuxi-Know 的两个核心操作封装为标准 Claude Code Skills，供其他 Agent 复用：

- **`yuxi-upload`**：文档上传（上传 → 解析 → 入库三阶段）
- **`yuxi-rag`**：RAG 语义检索

---

## 2. 目录结构

```
.claude/skills/
├── yuxi-upload/
│   ├── SKILL.md                  # 主 skill 文件（执行流程 + 调用指令）
│   ├── scripts/
│   │   └── upload.py             # 三阶段上传逻辑 + 任务轮询
│   └── references/
│       └── api-spec.md           # 接口速查 + 错误码说明
│
└── yuxi-rag/
    ├── SKILL.md                  # 主 skill 文件
    ├── scripts/
    │   └── query.py              # 检索 + 结果格式化输出
    └── references/
        └── search-modes.md       # 检索模式选择指南
```

---

## 3. 配置与凭据管理

### 3.1 配置来源

所有凭据和运行时状态统一存储在 `.agentkit/config.yaml`，在现有文件中新增 `yuxi_know` 配置节：

```yaml
# .agentkit/config.yaml（新增节）
yuxi_know:
  base_url: "http://localhost:5050"   # Yuxi-Know API 地址
  username: "admin"                   # 登录用户名（对应 YUXI_SUPER_ADMIN_NAME）
  password: "your_password"           # 登录密码（对应 YUXI_SUPER_ADMIN_PASSWORD）

  # 运行时 token 缓存（由脚本自动维护，无需手动填写）
  _token_cache:
    access_token: ""
    expires_at: ""                    # ISO 8601 格式，用于判断是否过期
```

### 3.2 Token 缓存机制

- 脚本首次调用时读取 `_token_cache.access_token`
- 若 token 不存在或 `expires_at` 已过期（留 60 秒提前量），重新调用 `POST /api/auth/token` 获取新 token
- 获取后将 `access_token` 和 `expires_at` 写回 `.agentkit/config.yaml` 的 `_token_cache` 节
- 两个脚本（`upload.py` / `query.py`）共用同一缓存，通过 YAML 文件跨调用复用
- 遇到 401 响应时强制刷新 token 并重试一次

### 3.3 配置读取优先级

```
1. .agentkit/config.yaml 中的 yuxi_know 节（主要来源）
2. 环境变量（YUXI_BASE_URL / YUXI_USERNAME / YUXI_PASSWORD）作为备选覆盖
```

---

## 4. `yuxi-upload` Skill

### 4.1 调用接口

```
/yuxi-upload --file <path> | --url <url>
             --db <db_id | db_name>
             [--folder <folder_name>]
             [--no-index]
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `--file` | string | 二选一 | 本地文件路径 |
| `--url` | string | 二选一 | 远程文件 URL |
| `--db` | string | 是 | 知识库 ID 或知识库**名称** |
| `--folder` | string | 否 | 目标文件夹名称，不传则上传到根目录 |
| `--no-index` | flag | 否 | 仅解析，不自动入库（`auto_index: false`） |

### 4.2 执行流程

```
Step 0: 鉴权
        读取 .agentkit/config.yaml 的 _token_cache
        若有效 → 直接使用；若过期或空 → POST /api/auth/token → 写回缓存
        ↓
Step 1: 解析知识库参数
        若 --db 看起来像 UUID → 直接作为 db_id
        否则 → GET /api/knowledge/databases 获取列表，按 name 精确匹配
        若匹配失败 → 进入「知识库引导流程」（见 4.3）
        ↓
Step 2: 解析目录参数（仅 --folder 有值时）
        GET /api/knowledge/databases/{db_id} 获取文件列表
        过滤 is_folder=true，按 filename 匹配 --folder 值
        若存在 → 取 file_id 作为 parent_id
        若不存在 → POST /databases/{db_id}/folders 创建，取返回的 file_id
        ↓
Step 3: 上传内容
        --file → POST /api/knowledge/files/upload?db_id=xxx （multipart/form-data）
        --url  → POST /api/knowledge/files/fetch-url
        获得 file_path（MinIO URL）+ content_hash
        ↓
Step 4: 添加到知识库
        POST /api/knowledge/databases/{db_id}/documents
        Body:
          items: [file_path]
          params:
            content_hashes: { file_path: content_hash }  # MinIO 文件时必填
            parent_id: <folder file_id>                  # 可选
            auto_index: true | false
        获得 task_id
        ↓
Step 5: 轮询任务状态（auto_index=true 时执行）
        GET /api/tasks/{task_id}  每 3 秒轮询一次，超时上限 300s
        status=done    → 成功，输出耗时
        status=failed  → 报错，打印 task 详情
        status=超时    → 返回 task_id，提示用户手动查询
```

### 4.3 知识库引导流程（--db 名称未匹配时）

```
❌ 未找到名称为 "XXX" 的知识库

可用的知识库列表：
  1. 产品文档库        (db_id: abc123)
  2. 技术规范库        (db_id: def456)
  3. 客服知识库        (db_id: ghi789)

请使用以上 db_id 或精确名称重新调用，例如：
  /yuxi-upload --file ./doc.pdf --db abc123
```

### 4.4 输出示例

**成功：**
```
✅ 上传完成
   知识库: 产品文档库 (db_id: abc123)
   文件:   report.pdf
   目录:   技术规范/ (parent_id: folder_xyz)
   file_id: file_aaa111
   入库状态: done（耗时 23s）
```

**重复文件（409）：**
```
⚠️  文件已存在（相同内容）
   file_id: file_aaa111
   提示: 若需重新索引，请手动触发 /api/knowledge/databases/{db_id}/documents/index
```

### 4.5 错误处理

| 场景 | 处理方式 |
|------|----------|
| 401 Unauthorized | 强制刷新 token，重试一次 |
| 409 Conflict（重复文件） | 友好提示 + 返回已有 file_id |
| task status: failed | 打印 task 错误详情，建议检查文件格式 |
| 任务超时（>300s） | 返回 task_id，提示用户执行 `GET /api/tasks/{task_id}` 手动确认 |
| --db 名称未匹配 | 列出可用知识库列表，引导重新选择（见 4.3） |

---

## 5. `yuxi-rag` Skill

### 5.1 调用接口

```
/yuxi-rag --query <text>
          --db <db_id | db_name>
          [--top-k 5]
          [--mode hybrid]
          [--threshold 0.2]
          [--file <filename>]
```

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `--query` | string | 必填 | 检索问题 |
| `--db` | string | 必填 | 知识库 ID 或知识库名称 |
| `--top-k` | int | `5` | 返回结果数量 |
| `--mode` | string | `hybrid` | 检索模式：`vector` / `keyword` / `hybrid` |
| `--threshold` | float | `0.2` | 相似度过滤阈值（0~1） |
| `--file` | string | - | 按文件名过滤，支持模糊匹配 |

### 5.2 执行流程

```
Step 0: 鉴权（同 upload，读取/刷新 .agentkit/config.yaml _token_cache）
        ↓
Step 1: 解析知识库参数
        逻辑与 yuxi-upload Step 1 完全一致
        名称未匹配时同样进入「知识库引导流程」
        ↓
Step 2: 执行检索
        POST /api/knowledge/databases/{db_id}/query
        Body:
          query: <--query 值>
          meta:
            final_top_k: <--top-k>
            search_mode: <--mode>
            similarity_threshold: <--threshold>
            file_name: <--file>（可选）
        ↓
Step 3: 格式化输出结果
```

### 5.3 检索模式选择建议

| 场景 | 推荐模式 |
|------|----------|
| 语义理解、自然语言问答 | `hybrid`（默认）|
| 精确词匹配、代码/型号检索 | `keyword` |
| 纯向量相似度（无关键词） | `vector` |

### 5.4 输出示例

```
🔍 检索: "产品退款政策" | 知识库: 产品文档库 | 模式: hybrid | Top-5

─────────────────────────────────────────────
1. [相似度 0.92]  退款政策.pdf  §第3章-退款流程
   "退款申请需在购买后7天内提交，支持原路退回或账户余额..."

2. [相似度 0.87]  FAQ.docx  §常见问题
   "超过7天的退款需联系客服，由客服人工审核处理..."

3. [相似度 0.74]  运营手册.pdf  §第5章
   "特殊情况下（商品质量问题），可申请超期退款..."
─────────────────────────────────────────────
共返回 3 条（threshold=0.2 过滤后）
```

**名称未匹配时（同 upload）：**
```
❌ 未找到名称为 "产品库" 的知识库

可用的知识库列表：
  1. 产品文档库  (db_id: abc123)
  2. 技术规范库  (db_id: def456)
  ...
```

---

## 6. 公共模块设计

### 6.1 共用工具函数（两个 script 均引用）

```python
# 伪代码示意，实际在各脚本内实现
def load_config() -> dict           # 读取 .agentkit/config.yaml
def save_config(config: dict)       # 写回 .agentkit/config.yaml
def get_token(config: dict) -> str  # 读缓存 or 重新登录 + 写缓存
def resolve_db_id(token, base_url, db_input) -> str  # 名称→ID，失败则打印列表并退出
```

### 6.2 `.agentkit/config.yaml` 完整新增示例

```yaml
# 在现有 config.yaml 末尾追加以下内容

yuxi_know:
  base_url: "http://localhost:5050"
  username: "admin"
  password: "your_password"

  _token_cache:
    access_token: ""
    expires_at: ""
```

---

## 7. Skill 间协作关系

```
其他 Agent
    │
    ├─ /yuxi-upload ──写入知识──→ Yuxi-Know API
    │                               ↑ (三阶段: upload → parse → index)
    │
    └─ /yuxi-rag    ──语义检索──→ Yuxi-Know API
                                    ↓ (结构化结果返回 Agent)
```

两个 skill 相互独立，均通过 `.agentkit/config.yaml` 共享 token 缓存，无其他依赖。

---

## 8. 文件清单

| 文件 | 说明 | 预计行数 |
|------|------|----------|
| `.claude/skills/yuxi-upload/SKILL.md` | 主 skill，描述触发条件、参数解析、执行步骤、输出格式 | ~100 行 |
| `.claude/skills/yuxi-upload/scripts/upload.py` | 三阶段上传 + token 管理 + 任务轮询 | ~200 行 |
| `.claude/skills/yuxi-upload/references/api-spec.md` | 接口速查表、错误码、注意事项 | ~60 行 |
| `.claude/skills/yuxi-rag/SKILL.md` | 主 skill，描述触发条件、参数解析、执行步骤、输出格式 | ~80 行 |
| `.claude/skills/yuxi-rag/scripts/query.py` | 检索 + token 管理 + 格式化输出 | ~120 行 |
| `.claude/skills/yuxi-rag/references/search-modes.md` | 检索模式选择指南 | ~40 行 |

---

## 9. 实现顺序建议

1. `upload.py` 中的公共函数（token 管理、db 解析、config 读写）
2. `yuxi-upload` SKILL.md + `upload.py` 完整实现
3. `yuxi-rag` SKILL.md + `query.py`（复用 token/db 逻辑）
4. 联调测试：上传 → 等待入库 → 检索验证
