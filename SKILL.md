---
name: mediary
description: 操作 Mediary 日记后端 API 的技能手册。用于 AI Agent 对接 Mediary 实现文档池读写，支持 API Key 鉴权、文档 CRUD、标签管理、搜索过滤。
---

# Mediary Skill

## Purpose

提供一份操作 Mediary 日记后端 API 的完整操作手册，使 AI Agent 能够通过 API Key 认证后自主完成文档读写、标签管理、搜索过滤等任务。

## When To Use

- AI Agent 需要读取/写入用户日记文档
- 将 Mediary 作为知识库被其他工具调用
- 自动化文档整理、标签批处理、定时归档

## Quick Start

### 1. 获取 API Key

登录 Mediary Web UI → 进入「设备」页面 →「AI 接口密钥」tab → 点击「生成 API Key」→ 复制保存（明文只显示一次）

### 2. 配置

使用前，必须先确认用户的 Mediary 后端地址和 API Key。通过对话向用户询问：

- **BASE_URL**：用户的 Mediary 后端地址（如 `https://example.com/api/v1`）
- **API_KEY**：用户在 Mediary 前端生成的 API Key

### 3. 验证连接

```bash
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
  "YOUR_BASE_URL/tags" | head -20
```

返回 `{"code":0,"data":[...],"message":"ok"}` 即为成功。

## 请求格式

所有请求需要携带认证 Header：

```
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
```

## Core API Reference

> 下面用 `BASE_URL` 代替用户的实际后端地址。

### 文档操作

#### List 获取文档列表
```
GET BASE_URL/documents?page=1&limit=20&type=diary&q=keyword&search_mode=title_only
```

参数：
- `page` / `limit`：分页
- `type`：`diary` | `note` | `topic`，不传则全部
- `q`：关键词搜索
- `search_mode`：`title_only` | `content_only`，默认搜全部

#### Get 获取单个文档
```
GET BASE_URL/documents/:id
```

#### Create 创建文档
```
POST BASE_URL/documents
Body: {
  "title": "标题",
  "content": "正文内容",
  "doc_type": "diary|note|topic",
  "source": "ai",
  "created_at": "2024-01-01",
  "tags": ["tag1", "tag2"]
}
```

#### Update 更新文档
```
PUT BASE_URL/documents/:id
Body: {
  "title": "新标题",
  "content": "新正文",
  "doc_type": "note",
  "tags": ["tag1"]
}
```

#### Delete 移到回收站
```
DELETE BASE_URL/documents/:id
```

#### Search 搜索文档
```
GET BASE_URL/documents/search?q=keyword&tag=tag1&type=diary&start=2024-01-01&end=2024-12-31&search_mode=title_only&page=1&limit=20
```

#### BulkAddTags 批量添加标签
```
POST BASE_URL/documents/bulk-add-tags
Body: {"doc_ids": [1,2,3], "tags": ["tag1","tag2"]}
```

#### BulkRemoveTags 批量移除标签
```
POST BASE_URL/documents/bulk-remove-tags
Body: {"doc_ids": [1,2,3], "tags": ["tag1","tag2"]}
```

### 标签操作

#### List 获取所有标签
```
GET BASE_URL/tags
```

#### Create 创建标签
```
POST BASE_URL/tags
Body: {"name": "标签名", "color": "#FF0000"}
```

#### Delete 删除标签
```
DELETE BASE_URL/tags/:name
```

#### Stats 获取标签使用统计
```
GET BASE_URL/tags/stats
```

## Response Format

所有接口返回统一 JSON 格式：

```json
// 成功
{"code": 0, "message": "ok", "data": {...}}

// 错误
{"code": 40101, "message": "Authorization header is required", "data": null}
```

分页列表返回：
```json
{"code": 0, "message": "ok", "data": [...], "total": 100}
```

## doc_type 枚举

| 值 | 含义 |
|---|---|
| `diary` | 日记 |
| `note` | 笔记 |
| `topic` | 专题 |

## Error Handling

| 场景 | 处理方式 |
|------|---------|
| 40101 | API Key 无效或未提供，检查 Key 是否正确 |
| 404 | 文档/标签不存在 |
| 400 | 请求参数错误，检查 Body 格式 |
| 500 | 服务器错误，查看后端日志 |

## Security Notes

- API Key 明文只显示一次，生成后请立即保存
- Key 存储为 bcrypt hash，丢失需撤销后重新生成
- 不要在代码仓库中硬编码 API Key

## Limitations

- API Key 对所有端点有效，暂无细粒度权限控制
- 文档内容无版本管理，覆盖更新需自行备份
- 搜索为模糊匹配，无全文索引
