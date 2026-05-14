# Mediary API Examples

> 使用前替换 `BASE_URL` 和 `YOUR_TOKEN` 为实际值。Token 通过登录接口获取（见 SKILL.md 认证流程）。

## 认证示例

### 发送验证码
```bash
curl -s -X POST "BASE_URL/../api/auth/send-code" \
  -H "Content-Type: application/json" -d '{}'
```

### 登录获取 Token
```bash
curl -s -X POST "BASE_URL/../api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"code":"123456","login_type":"email","device_id":"agent-001","device_name":"AI Agent"}'
```

## 文档 CRUD 示例

### 获取文档列表
```bash
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents?page=1&limit=5"
```

### 搜索文档
```bash
# 关键词搜索
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/search?q=AI&type=note&limit=10"

# 按标签搜索
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/search?tag=工作&page=1&limit=50"

# 按日期范围搜索
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/search?start=2026-01-01&end=2026-01-31&limit=50"

# 仅搜标题
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/search?q=会议&search_mode=title_only"

# 置顶筛选
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/search?is_pinned=true"
```

### 按日期获取文档
```bash
# ⚠️ 参数是 start 和 end，不是 date
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/by-date?start=2026-05-14&end=2026-05-14"
```

### 创建文档
```bash
curl -s -X POST "BASE_URL/documents" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "示例文档",
    "content": "正文内容",
    "doc_type": "note",
    "source": "local",
    "tags": ["AI", "草稿"]
  }'
```

> ⚠️ `source` 只能是 `notion`/`feishu`/`local`，不要传 `"ai"`。

### 更新文档
```bash
# 全量更新
curl -s -X PUT "BASE_URL/documents/123" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "新标题", "content": "新内容"}'

# 部分更新（推荐）
curl -s -X PATCH "BASE_URL/documents/123/patch" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"title": "只更新标题"}'
```

### 删除与恢复
```bash
# 移到回收站
curl -s -X DELETE -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/123"

# 从回收站恢复
curl -s -X POST -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/123/restore"

# 永久删除
curl -s -X DELETE -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/123/permanent"

# 获取回收站列表
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/trash"
```

### 置顶操作
```bash
# 切换置顶（⚠️ 路径是 /pin，不是 /toggle-pin）
curl -s -X POST -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/123/pin"

# 获取置顶文档
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/documents/pinned"
```

### 批量操作
```bash
# 批量添加标签
curl -s -X POST "BASE_URL/documents/bulk-add-tags" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"doc_ids": [1, 2, 3], "tags": ["AI"]}'

# 批量移除标签
curl -s -X POST "BASE_URL/documents/bulk-remove-tags" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"doc_ids": [1, 2, 3], "tags": ["AI"]}'

# 批量删除
curl -s -X POST "BASE_URL/documents/bulk-delete" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"ids": [1, 2, 3]}'
```

## 标签示例

```bash
# 列出所有标签
curl -s -H "Authorization: Bearer YOUR_TOKEN" "BASE_URL/tags"

# 创建标签
curl -s -X POST "BASE_URL/tags" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "新标签", "color": "#FF5733"}'

# 删除标签（⚠️ 用数字 ID，不是标签名）
curl -s -X DELETE -H "Authorization: Bearer YOUR_TOKEN" "BASE_URL/tags/42"

# 标签统计
curl -s -H "Authorization: Bearer YOUR_TOKEN" "BASE_URL/tags/stats"
```

## 块级编辑示例

```bash
# 获取文档的所有块
curl -s -H "Authorization: Bearer YOUR_TOKEN" "BASE_URL/blocks/document/123"

# 替换文档的所有块
curl -s -X PUT "BASE_URL/blocks/document/123" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[{"type":"paragraph","content":"第一段"},{"type":"heading","content":"标题","level":1}]'
```

## Python 示例

```python
import requests

TOKEN = "YOUR_TOKEN"
BASE_URL = "YOUR_BASE_URL"

headers = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json"
}

def search_documents(keyword, doc_type="note", is_pinned=None):
    params = {"q": keyword, "type": doc_type}
    if is_pinned is not None:
        params["is_pinned"] = str(is_pinned).lower()
    resp = requests.get(f"{BASE_URL}/documents/search", params=params, headers=headers)
    return resp.json()

def get_all_documents(limit=100):
    all_docs = []
    page = 1
    while True:
        resp = requests.get(
            f"{BASE_URL}/documents",
            params={"page": page, "limit": limit},
            headers=headers
        ).json()
        docs = resp.get("data", [])
        total = resp.get("total", 0)
        all_docs.extend(docs)
        if len(all_docs) >= total or not docs:
            break
        page += 1
    return all_docs

def create_document(title, content, doc_type="note", tags=None):
    payload = {
        "title": title,
        "content": content,
        "doc_type": doc_type,
        "source": "local",  # ⚠️ 只能是 notion/feishu/local
        "tags": tags or []
    }
    resp = requests.post(f"{BASE_URL}/documents", json=payload, headers=headers)
    return resp.json()

def patch_document(doc_id, **fields):
    resp = requests.patch(
        f"{BASE_URL}/documents/{doc_id}/patch",
        json=fields, headers=headers
    )
    return resp.json()

def toggle_pin(doc_id):
    resp = requests.post(f"{BASE_URL}/documents/{doc_id}/pin", headers=headers)
    return resp.json()

def delete_document(doc_id):
    resp = requests.delete(f"{BASE_URL}/documents/{doc_id}", headers=headers)
    return resp.json()

def restore_document(doc_id):
    resp = requests.post(f"{BASE_URL}/documents/{doc_id}/restore", headers=headers)
    return resp.json()

def get_documents_by_date(start_date, end_date=None):
    params = {"start": start_date}  # ⚠️ 参数是 start，不是 date
    if end_date:
        params["end"] = end_date
    resp = requests.get(f"{BASE_URL}/documents/by-date", params=params, headers=headers)
    return resp.json()

def bulk_add_tags(doc_ids, tags):
    resp = requests.post(
        f"{BASE_URL}/documents/bulk-add-tags",
        json={"doc_ids": doc_ids, "tags": tags},
        headers=headers
    )
    return resp.json()

def delete_tag(tag_id):
    resp = requests.delete(f"{BASE_URL}/tags/{tag_id}", headers=headers)
    return resp.json()

def safe_api_call(method, url, **kwargs):
    """统一错误处理"""
    resp = getattr(requests, method)(url, headers=headers, **kwargs)
    data = resp.json()
    if data["code"] == 0:
        return data["data"]
    elif data["code"] == 40103:
        print("Token 过期，请重新登录")
    elif data["code"] == 40301:
        print("设备已被撤销")
    else:
        print(f"API 错误: {data['message']}")
    return None
```
