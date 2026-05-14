---
name: mediary
description: 操作 Mediary 日记后端 API 的技能手册。用于 AI Agent 对接 Mediary 实现文档池读写，支持 JWT 鉴权、文档 CRUD、标签管理、搜索过滤。
---

# Mediary Skill

## Purpose

提供一份操作 Mediary 日记后端 API 的完整操作手册，使 AI Agent 能够通过 JWT 认证后自主完成文档读写、标签管理、搜索过滤等任务。

## When To Use

- AI Agent 需要读取/写入用户日记文档
- 将 Mediary 作为知识库被其他工具调用
- 自动化文档整理、标签批处理、定时归档

## Quick Start

### 1. 配置

使用前，必须先确认用户的 Mediary 后端地址：

- **BASE_URL**：用户的 Mediary 后端地址（如 `https://riji.7ygv.com/api/v1`）

### 2. 认证流程（JWT Token）

Mediary 使用 JWT Token 认证，**不是 API Key**。获取 Token 的流程：

**方式一：邮箱验证码登录**
```bash
# Step 1: 发送验证码到管理员邮箱
curl -s -X POST "BASE_URL/../api/auth/send-code" \
  -H "Content-Type: application/json" -d '{}'
# 返回 {"code":0,"message":"ok","data":"验证码已发送"}

# Step 2: 用验证码登录获取 Token（需要用户从邮箱获取验证码）
curl -s -X POST "BASE_URL/../api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"code":"123456","login_type":"email","device_id":"agent-001","device_name":"AI Agent"}'
# 返回 {"code":0,"data":{"token":"eyJhbG..."},"message":"ok"}
```

**方式二：TOTP 登录**
```bash
curl -s -X POST "BASE_URL/../api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"code":"123456","login_type":"totp","device_id":"agent-001","device_name":"AI Agent"}'
```

> Token 有效期 7 天。过期后需重新登录获取。

### 3. 验证连接

```bash
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/tags" | python3 -m json.tool
```

返回 `{"code":0,"data":[...],"message":"ok"}` 即为成功。

## 请求格式

所有请求需要携带认证 Header：

```
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json
```

## Core API Reference

> 下面用 `BASE_URL` 代替用户的实际后端地址（如 `https://riji.7ygv.com/api/v1`）。

### 文档操作

#### List 获取文档列表
```
GET BASE_URL/documents?page=1&limit=20&type=diary&q=keyword&search_mode=title_only&is_pinned=true
```

参数：
- `page` / `limit`：分页（默认 page=1, limit=20，最大 limit=2000）
- `type`：`diary` | `note` | `topic`，不传则全部
- `q`：关键词搜索
- `search_mode`：`title_only` | `content_only`，默认搜全部
- `is_pinned`：`true` 仅置顶文档，`false` 仅非置顶，不传则全部

#### Get 获取单个文档
```
GET BASE_URL/documents/:id
```

返回包含 `content` 字段（纯文本内容）。

#### Create 创建文档
```
POST BASE_URL/documents
Body: {
  "title": "标题",
  "content": "正文内容",
  "doc_type": "diary|note|topic",
  "source": "local",
  "tags": ["tag1", "tag2"]
}
```

**⚠️ `source` 字段枚举值：** `notion` | `feishu` | `local`（默认 `local`）。不要传 `"ai"` 或其他值，会导致数据库报错。

#### Update 更新文档（全量覆盖）
```
PUT BASE_URL/documents/:id
Body: {
  "title": "新标题",
  "content": "新正文",
  "doc_type": "note",
  "tags": ["tag1"]
}
```

#### Patch 更新文档（部分字段）
```
PATCH BASE_URL/documents/:id/patch
Body: {
  "title": "只更新标题"
}
```

#### Delete 移到回收站
```
DELETE BASE_URL/documents/:id
```

#### Restore 从回收站恢复
```
POST BASE_URL/documents/:id/restore
```

#### PermanentDelete 永久删除
```
DELETE BASE_URL/documents/:id/permanent
```

#### ListTrash 获取回收站列表
```
GET BASE_URL/documents/trash?page=1&limit=20
```

#### Search 搜索文档（组合筛选）
```
GET BASE_URL/documents/search?q=keyword&tag=tag1&type=diary&start=2024-01-01&end=2024-12-31&search_mode=title_only&is_pinned=true&page=1&limit=20
```

参数：
- `q`：关键词（模糊匹配标题和内容）
- `tag`：标签名（精确匹配，多个用逗号分隔）
- `type`：文档类型
- `start` / `end`：日期范围（格式 YYYY-MM-DD）
- `search_mode`：`title_only` 仅搜标题
- `is_pinned`：`true` 仅置顶，`false` 仅非置顶
- `page` / `limit`：分页

#### ByDate 按日期获取文档
```
GET BASE_URL/documents/by-date?start=2024-01-01&end=2024-01-31&page=1&limit=20
```

**⚠️ 参数是 `start` 和 `end`，不是 `date`。**

#### ByTag 按标签获取文档
```
GET BASE_URL/documents/by-tag?tag=标签名&page=1&limit=20
```

#### TogglePin 切换置顶状态
```
POST BASE_URL/documents/:id/pin
```

**⚠️ 路径是 `/pin`，不是 `/toggle-pin`。**

#### GetPinned 获取所有置顶文档
```
GET BASE_URL/documents/pinned
```

#### GetTitles 批量获取文档标题
```
POST BASE_URL/documents/titles
Body: {"ids": [1, 2, 3]}
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

#### BulkDelete 批量删除
```
POST BASE_URL/documents/bulk-delete
Body: {"ids": [1, 2, 3]}
```

### 块级编辑（Block）

#### GetByDocID 获取文档的所有块
```
GET BASE_URL/blocks/document/:doc_id
```

#### ReplaceAll 替换文档的所有块
```
PUT BASE_URL/blocks/document/:doc_id
Body: [
  {"type": "paragraph", "content": "第一段"},
  {"type": "heading", "content": "标题", "level": 1}
]
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
DELETE BASE_URL/tags/:id
```

**⚠️ 删除用的是标签的数字 ID，不是标签名。**

#### Stats 获取标签使用统计
```
GET BASE_URL/tags/stats
```

### 共享文档（无需认证）

#### GetShared 获取共享文档
```
GET /api/shared/documents/:id
```

无需 Authorization header。文档需先在前端设置为"共享"状态才能访问。

### Todo 操作

#### List 获取 Todo 列表
```
GET BASE_URL/todos?page=1&limit=20&status=doing
```

#### Create 创建 Todo
```
POST BASE_URL/todos
Body: {"title": "任务标题", "status": "todo", "priority": 1}
```

#### Get 获取单个 Todo
```
GET BASE_URL/todos/:id
```

#### Update 更新 Todo
```
PUT BASE_URL/todos/:id
Body: {"title": "新标题", "status": "done"}
```

#### Delete 删除 Todo
```
DELETE BASE_URL/todos/:id
```

#### GetFullNetwork 获取 Todo 依赖关系网络
```
GET BASE_URL/todos/network
```

#### AddDependency 添加依赖关系
```
POST BASE_URL/todos/dependency
Body: {"todo_id": 1, "depends_on": 2}
```

#### RemoveDependency 移除依赖关系
```
DELETE BASE_URL/todos/dependency
Body: {"todo_id": 1, "depends_on": 2}
```

### Todo 标签

#### ListTags 获取 Todo 标签列表
```
GET BASE_URL/todo-tags
```

#### CreateTag 创建 Todo 标签
```
POST BASE_URL/todo-tags
Body: {"name": "标签名", "color": "#00FF00"}
```

#### UpdateTag 更新 Todo 标签
```
PUT BASE_URL/todo-tags/:id
Body: {"name": "新名"}
```

#### DeleteTag 删除 Todo 标签
```
DELETE BASE_URL/todo-tags/:id
```

### 番茄钟（Tomatoes）

#### ListByDate 按日期获取番茄钟记录
```
GET BASE_URL/tomatoes?date=2024-01-01
```

#### Upsert 创建/更新番茄钟记录
```
POST BASE_URL/tomatoes
Body: {"date": "2024-01-01", "count": 4, "duration": 100}
```

#### Update 更新番茄钟记录
```
PUT BASE_URL/tomatoes/:id
Body: {"count": 5}
```

#### Delete 删除番茄钟记录
```
DELETE BASE_URL/tomatoes/:id
```

#### GetSettings 获取番茄钟设置
```
GET BASE_URL/tomatoes/settings
```

#### SaveSettings 保存番茄钟设置
```
PUT BASE_URL/tomatoes/settings
Body: {"work_duration": 25, "break_duration": 5}
```

#### Stats 获取番茄钟统计
```
GET BASE_URL/tomatoes/stats
```

#### Heatmap 获取番茄钟热力图数据
```
GET BASE_URL/tomatoes/heatmap
```

### 账单（Expenses）

#### List 获取账单列表
```
GET BASE_URL/expenses?page=1&limit=20&start=2024-01-01&end=2024-12-31
```

#### Create 创建账单
```
POST BASE_URL/expenses
Body: {"amount": 50.5, "category": "餐饮", "description": "午餐", "date": "2024-01-01"}
```

#### Update 更新账单
```
PUT BASE_URL/expenses/:id
Body: {"amount": 60}
```

#### Delete 删除账单
```
DELETE BASE_URL/expenses/:id
```

#### Stats 获取账单统计
```
GET BASE_URL/expenses/stats?start=2024-01-01&end=2024-12-31
```

### 账号管理（Accounts）

#### List 获取账号列表
```
GET BASE_URL/accounts
```

#### Create 创建账号
```
POST BASE_URL/accounts
Body: {"platform": "GitHub", "username": "wind", "url": "https://github.com/wind"}
```

#### Update 更新账号
```
PUT BASE_URL/accounts/:id
Body: {"username": "newname"}
```

#### Delete 删除账号
```
DELETE BASE_URL/accounts/:id
```

### 目标管理（Goals）

#### List 获取目标列表
```
GET BASE_URL/goals
```

#### Create 创建目标
```
POST BASE_URL/goals
Body: {"title": "目标标题", "description": "描述", "deadline": "2024-12-31"}
```

#### Get 获取单个目标
```
GET BASE_URL/goals/:id
```

#### Update 更新目标
```
PUT BASE_URL/goals/:id
Body: {"title": "新标题"}
```

#### Delete 删除目标
```
DELETE BASE_URL/goals/:id
```

### 注意力记录（Attention）

#### ListPlatforms 获取平台列表
```
GET BASE_URL/attention/platforms
```

#### CreatePlatform 创建平台
```
POST BASE_URL/attention/platforms
Body: {"name": "YouTube", "url": "https://youtube.com"}
```

#### UpdatePlatform 更新平台
```
PUT BASE_URL/attention/platforms/:id
```

#### DeletePlatform 删除平台
```
DELETE BASE_URL/attention/platforms/:id
```

#### ListDevices 获取设备列表
```
GET BASE_URL/attention/devices
```

#### CreateDevice 创建设备
```
POST BASE_URL/attention/devices
Body: {"name": "iPhone"}
```

#### UpdateDevice 更新设备
```
PUT BASE_URL/attention/devices/:id
```

#### DeleteDevice 删除设备
```
DELETE BASE_URL/attention/devices/:id
```

#### LinkDevicePlatform 关联设备和平台
```
POST BASE_URL/attention/link/device/:deviceID/platform/:platformID
```

#### UnlinkDevicePlatform 取消关联
```
DELETE BASE_URL/attention/link/device/:deviceID/platform/:platformID
```

### 偏好设置（Preferences）

#### Get 获取偏好
```
GET BASE_URL/preferences/:key
```

#### Set 设置偏好
```
POST BASE_URL/preferences
Body: {"key": "theme", "value": "dark"}
```

### 日记总结（Summaries）

#### Generate 生成总结
```
POST BASE_URL/summaries/generate
Body: {"date": "2024-01-01"}
```

#### Get 获取总结
```
GET BASE_URL/summaries/:date
```

### 设备管理

#### ListDevices 获取已注册设备
```
GET BASE_URL/devices
```

### TOTP 管理

#### GetUserTOTPStatus 获取 TOTP 状态
```
GET BASE_URL/totp/status
```

#### SetUserTOTP 设置 TOTP
```
POST BASE_URL/totp/setup
Body: {"secret": "BASE32SECRET"}
```

#### RemoveUserTOTP 移除 TOTP
```
DELETE BASE_URL/totp/remove
```

## Response Format

所有接口返回统一 JSON 格式：

```json
// 成功
{"code": 0, "message": "ok", "data": {...}}

// 业务错误
{"code": 1, "message": "具体错误信息", "data": null}

// 认证错误
{"code": 40101, "message": "Authorization header is required", "data": null}
{"code": 40102, "message": "Authorization header format must be Bearer {token}", "data": null}
{"code": 40103, "message": "Invalid or expired token", "data": null}

// 设备黑名单
{"code": 40301, "message": "设备已被撤销，请重新验证", "data": {"device_id": "...", "can_reauth": true}}
```

分页列表返回：
```json
{"code": 0, "message": "ok", "data": [...], "total": 100}
```

## 分页策略

默认每页返回 20 条，最多 2000 条。需要获取完整列表时，必须遍历所有页面：

```bash
# 第1页
curl -s -H "Authorization: Bearer TOKEN" "BASE_URL/documents?page=1&limit=100"
# 返回 {"code":0,"data":[...],"total":350}

# 根据 total 计算总页数: ceil(350/100) = 4
# 继续请求 page=2,3,4
```

**注意**：`total` 字段在响应顶层，表示匹配的文档总数。当 `data` 返回空数组或达到总页数时停止遍历。

## doc_type 枚举

| 值 | 含义 |
|---|---|
| `diary` | 日记 |
| `note` | 笔记 |
| `topic` | 专题 |

## source 枚举

| 值 | 含义 |
|---|---|
| `local` | 本地（默认，推荐使用） |
| `notion` | 从 Notion 导入 |
| `feishu` | 从飞书导入 |

**⚠️ 不要传 `"ai"` 或其他值，会导致数据库报错 `Data truncated for column 'source'`。**

## Error Handling

| 场景 | 处理方式 |
|------|---------|
| 40101 | 未提供 Authorization header |
| 40102 | Header 格式错误，必须是 `Bearer {token}` |
| 40103 | Token 无效或已过期，需重新登录 |
| 40301 | 设备已被撤销，需重新认证 |
| code=1 | 业务错误，查看 message 字段 |
| 500 | 服务器错误，查看后端日志 |

## Security Notes

- JWT Token 有效期 7 天，过期需重新登录
- 不要在代码仓库中硬编码 Token
- 设备被撤销后 Token 立即失效

## Document Bookmarks（文档书签）

对于需要反复读写的固定文档，可以用"书签"方式管理：把文档 ID 存入配置，通过 block API 按需读取前 N 个 block，避免读取全文。

### 工作原理

- 每个文档的内容以 blocks 形式存储，block 按 position 排序
- 新内容追加到文档顶部（position=0），旧内容自然下沉
- AI Agent 只需读取前 N 个 block 即可获取最新需求，不会读到旧的结构化内容

### 读取书签文档（前 N 个 block）

```bash
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/blocks/document/8387" | python3 -c "
import sys, json
blocks = json.load(sys.stdin).get('data', [])
for b in blocks[:5]:  # 只取前5个
    print(f'[{b[\"block_type\"]}] {b[\"content\"][:200]}')
"
```

### 追加内容到书签文档顶部

```bash
# 1. 先读取现有 blocks
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/blocks/document/8387" > /tmp/existing_blocks.json

# 2. 在 Python 中构造新的 blocks 列表（新内容在前，旧内容在后）
python3 -c "
import json
existing = json.load(open('/tmp/existing_blocks.json'))['data']
new_blocks = [
    {'block_type': 'heading', 'content': '新需求标题', 'level': 3},
    {'block_type': 'list_item', 'content': '需求描述'},
]
all_blocks = new_blocks + existing
print(json.dumps(all_blocks))
" > /tmp/all_blocks.json

# 3. 替换所有 blocks
curl -s -X PUT "BASE_URL/blocks/document/8387" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/all_blocks.json
```

### 标记需求为 [done]

在需求文本前加 `[done]` 前缀即可：

```bash
# 找到目标 block，修改 content 后 PUT 回去
# 先读取 → 修改 → 替换全部 blocks
```

### 当前书签列表

| 名称 | 文档 ID | 用途 | 建议读取 block 数 |
|------|---------|------|-------------------|
| mediary-dev-plan | 8387 | Mediary 开发需求计划 | 前 15 个（获取最新待做需求） |

> 用户可通过对话告诉 Agent 新增/修改书签。Agent 应在 SKILL.md 的书签表中维护。

## Limitations

- Token 对所有端点有效，暂无细粒度权限控制
- 文档内容无版本管理，覆盖更新需自行备份
- 搜索为模糊匹配，无全文索引
- `source` 字段只能是 `notion`/`feishu`/`local`，不支持自定义值
