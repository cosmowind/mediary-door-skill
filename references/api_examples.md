# Mediary API Examples

## curl 示例

### 验证连接
```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  http://localhost:3000/api/v1/tags | jq .
```

### 创建文档
```bash
curl -X POST http://localhost:3000/api/v1/documents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "AI 辅助写作示例",
    "content": "这是正文内容...",
    "doc_type": "note",
    "source": "ai",
    "created_at": "2024-05-10",
    "tags": ["AI", "草稿"]
  }' | jq .
```

### 搜索文档
```bash
curl -s "http://localhost:3000/api/v1/documents/search?q=AI&type=note&limit=10" \
  -H "Authorization: Bearer YOUR_API_KEY" | jq .
```

### 批量添加标签
```bash
curl -X POST http://localhost:3000/api/v1/documents/bulk-add-tags \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"doc_ids": [1, 2, 3], "tags": ["AI"]}' | jq .
```

### 列出所有标签
```bash
curl -s http://localhost:3000/api/v1/tags \
  -H "Authorization: Bearer YOUR_API_KEY" | jq .
```

## Python 示例

```python
import requests

API_KEY = "YOUR_API_KEY"
BASE_URL = "http://localhost:3000/api/v1"
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

# 搜索文档
def search_documents(keyword, doc_type="note"):
    resp = requests.get(
        f"{BASE_URL}/documents/search",
        params={"q": keyword, "type": doc_type},
        headers=HEADERS
    )
    return resp.json()

# 创建文档
def create_document(title, content, doc_type="note", tags=None):
    payload = {
        "title": title,
        "content": content,
        "doc_type": doc_type,
        "source": "ai",
        "created_at": "2024-05-10",
        "tags": tags or []
    }
    resp = requests.post(f"{BASE_URL}/documents", json=payload, headers=HEADERS)
    return resp.json()

# 批量添加标签
def bulk_add_tags(doc_ids, tags):
    resp = requests.post(
        f"{BASE_URL}/documents/bulk-add-tags",
        json={"doc_ids": doc_ids, "tags": tags},
        headers=HEADERS
    )
    return resp.json()
```

## JavaScript / Node.js 示例

```javascript
const API_KEY = "YOUR_API_KEY";
const BASE_URL = "http://localhost:3000/api/v1";

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
      source: "ai", created_at: "2024-05-10", tags
    })
  });
  return resp.json();
}
```

## 错误处理示例

```python
import requests

def safe_api_call(fn, *args, **kwargs):
    try:
        result = fn(*args, **kwargs)
        if result.get("code") != 0:
            print(f"API Error: {result.get('msg')}")
        return result
    except requests.exceptions.RequestException as e:
        print(f"Network Error: {e}")
        return None
```

## 认证失败示例

```json
// 401 Unauthorized
{"code": 401, "msg": "Invalid or missing authentication token", "data": null}

// 重新获取 Key 后重试
```
