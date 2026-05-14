---
name: mediary
description: 操作 Mediary 日记后端 API 的技能手册。用于 AI Agent 对接 Mediary 实现文档池读写，支持 API Key 鉴权、文档 CRUD、标签管理、搜索过滤。
---

# Mediary Skill

## Purpose

提供一份操作 Mediary 日记后端 API 的完整操作手册，使 AI Agent 能够通过 API Key 认证后自主完成文档读写、标签管理、搜索过滤等任务。适用于文档池场景、AI 写作助手接入、个人知识库同步等用途。

## When To Use

- AI Agent 需要读取/写入用户日记文档
- 将 Mediary 作为知识库被其他工具调用
- 自动化文档整理、标签批处理、定时归档
- 通过脚本/API 批量操作文档

## Quick Start

### 1. 获取 API Key

登录 Mediary Web UI → 进入「身份验证」页面 → 点击「生成新密钥」→ 复制显示的明文 Key（一页只显示一次）

### 2. 配置请求

```bash
# 基础 URL
# dev 环境：https://dev.7ygv.com/api/v1  （后端端口 9000）
# 生产环境：https://riji.7ygv.com/api/v1  （后端端口 8081）
BASE_URL="https://dev.7ygv.com/api/v1"

# 请求 Header
-H "Authorization: Bearer YOUR_API_KEY"
-H "Content-Type: application/json"
```

### 3. 验证连接

```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  https://dev.7ygv.com/api/v1/tags | jq .
```

## Core API Reference

### 文档操作

#### List 获取文档列表
```
GET /api/v1/documents?page=1&limit=20&type=diary&q=keyword&search_mode=title_only
```

#### Get 获取单个文档
```
GET /api/v1/documents/:id
```

#### Create 创建文档
```
POST /api/v1/documents
Body: {"title":"标题","content":"正文","doc_type":"diary|note|topic","source":"local|ai","created_at":"2024-01-01","tags":["tag1","tag2"]}
```

#### Update 更新文档（全字段）
```
PUT /api/v1/documents/:id
Body: {"title":"新标题","content":"新正文","doc_type":"diary|note|topic","is_shared":true,"created_at":"2024-01-01","tags":["tag1"]}
```

#### Delete 移到回收站
```
DELETE /api/v1/documents/:id
```

#### Search 搜索文档
```
GET /api/v1/documents/search?q=keyword&tag=tag1&type=diary&start=2024-01-01&end=2024-12-31&search_mode=title_only&page=1&limit=20
```

#### BulkAddTags 批量添加标签
```
POST /api/v1/documents/bulk-add-tags
Body: {"doc_ids":[1,2,3],"tags":["tag1","tag2"]}
```

#### BulkRemoveTags 批量移除标签
```
POST /api/v1/documents/bulk-remove-tags
Body: {"doc_ids":[1,2,3],"tags":["tag1","tag2"]}
```

### 标签操作

#### List 获取所有标签
```
GET /api/v1/tags
```

#### Create 创建标签
```
POST /api/v1/tags
Body: {"name":"tag1","color":"#FF0000"}
```

#### Delete 删除标签
```
DELETE /api/v1/tags/:name
```

#### Stats 获取标签使用统计
```
GET /api/v1/tags/stats
```

### 认证操作

#### 生成 API Key
```
POST /api/v1/api-key
```
返回: `{"data":{"api_key":"<64位hex>","message":"API key 已生成，请妥善保管..."}}}`

#### 撤销 API Key
```
DELETE /api/v1/api-key
```

## Common Patterns

### 按日期范围查询日记
```
GET /api/v1/documents?type=diary&start=2024-01-01&end=2024-01-31&page=1&limit=50
```

### 搜索含特定标签的笔记
```
GET /api/v1/documents/search?tag=AI&type=note&limit=20
```

### AI 生成内容写入文档库
```
POST /api/v1/documents
{
  "title": "AI 辅助写作 - 主题标题",
  "content": "正文内容...",
  "doc_type": "note",
  "source": "ai",
  "created_at": "2024-05-10",
  "tags": ["AI", "草稿"]
}
```

### 批量为 AI 文档打标签
```
POST /api/v1/documents/bulk-add-tags
{"doc_ids":[10,11,12],"tags":["AI"]}
```

## Response Format

所有接口返回统一 JSON 格式：

```json
// 成功
{"code": 0, "msg": "success", "data": {...}}

// 错误
{"code": <HTTP状态码>, "msg": "错误描述", "data": null}
```

分页列表返回：
```json
{"code": 0, "msg": "success", "data": {
  "items": [...],
  "total": 100,
  "page": 1,
  "limit": 20
}}
```

## doc_type 枚举

| 值 | 含义 |
|---|---|
| `diary` | 日记 |
| `note` | 笔记 |
| `topic` | 专题 |

## search_mode 选项

| 值 | 含义 |
|---|---|
| `title_only` | 仅搜索标题 |
| `content_only` | 仅搜索正文 |
| 默认 | 标题+正文 |

## Error Handling

| 场景 | 处理方式 |
|---|---|
| 401 Unauthorized | API Key 无效或未提供，检查 Key 是否正确 |
| 404 Not Found | 文档/标签不存在 |
| 400 Bad Request | 请求参数错误，检查 Body 格式 |
| 500 Internal Error | 服务器错误，查看后端日志 |

## Security Notes

- API Key 明文只显示一次，生成后请立即保存
- Key 存储为 bcrypt hash，后端无法反查明文，丢失需撤销后重新生成
- 不要在代码仓库中硬编码 API Key，使用环境变量管理
- 定期撤销不再使用的 Key

## Limitations

- API Key 对所有 API 端点有效，暂无细粒度权限控制
- 文档内容无版本管理，覆盖更新需自行备份
- 回收站文档 30 天后自动清理，无单独恢复接口（PermanentDelete 除外）
- 搜索为模糊匹配，无全文索引优化

## File Reference

- `references/api_examples.md` — 完整 API 调用示例（curl / Python / JavaScript）
