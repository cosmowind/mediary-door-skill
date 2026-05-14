# Mediary API Examples

> 使用前替换 `YOUR_BASE_URL` 和 `YOUR_API_KEY` 为实际值。

## curl 示例

### 验证连接
```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  "YOUR_BASE_URL/tags" | jq .
```

### 获取文档列表
```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  "YOUR_BASE_URL/documents?page=1&limit=5" | jq .
```

### 创建文档
```bash
curl -s -X POST "YOUR_BASE_URL/documents" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "AI 辅助写作示例",
    "content": "这是正文内容...",
    "doc_type": "note",
    "source": "ai",
    "tags": ["AI", "草稿"]
  }' | jq .
```

### 搜索文档
```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  "YOUR_BASE_URL/documents/search?q=AI&type=note&limit=10" | jq .
```

### 获取单个文档
```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  "YOUR_BASE_URL/documents/123" | jq .
```

### 批量添加标签
```bash
curl -s -X POST "YOUR_BASE_URL/documents/bulk-add-tags" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"doc_ids": [1, 2, 3], "tags": ["AI"]}' | jq .
```

### 列出所有标签
```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  "YOUR_BASE_URL/tags" | jq .
```

## Python 示例

```python
import requests

API_KEY = "YOUR_API_KEY"
BASE_URL = "YOUR_BASE_URL"

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

# 搜索文档
def search_documents(keyword, doc_type="note"):
    resp = requests.get(
        f"{BASE_URL}/documents/search",
        params={"q": keyword, "type": doc_type},
        headers=headers
    )
    return resp.json()

# 创建文档
def create_document(title, content, doc_type="note", tags=None):
    payload = {
        "title": title,
        "content": content,
        "doc_type": doc_type,
        "source": "ai",
        "tags": tags or []
    }
    resp = requests.post(f"{BASE_URL}/documents", json=payload, headers=headers)
    return resp.json()

# 获取文档
def get_document(doc_id):
    resp = requests.get(f"{BASE_URL}/documents/{doc_id}", headers=headers)
    return resp.json()

# 批量添加标签
def bulk_add_tags(doc_ids, tags):
    resp = requests.post(
        f"{BASE_URL}/documents/bulk-add-tags",
        json={"doc_ids": doc_ids, "tags": tags},
        headers=headers
    )
    return resp.json()
```

## JavaScript / Node.js 示例

```javascript
const API_KEY = "YOUR_API_KEY";
const BASE_URL = "YOUR_BASE_URL";

const headers = {
  "Authorization": `Bearer ${API_KEY}`,
  "Content-Type": "application/json"
};

// 搜索文档
async function searchDocuments(keyword, docType = "note") {
  const resp = await fetch(
    `${BASE_URL}/documents/search?q=${encodeURIComponent(keyword)}&type=${docType}`,
    { headers }
  );
  return resp.json();
}

// 创建文档
async function createDocument(title, content, docType = "note", tags = []) {
  const resp = await fetch(`${BASE_URL}/documents`, {
    method: "POST",
    headers,
    body: JSON.stringify({
      title, content, doc_type: docType,
      source: "ai", tags
    })
  });
  return resp.json();
}

// 获取单个文档
async function getDocument(id) {
  const resp = await fetch(`${BASE_URL}/documents/${id}`, { headers });
  return resp.json();
}
```
