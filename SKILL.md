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

- **BASE_URL**：用户的 Mediary 后端地址（如 `https://your-domain.com/api/v1`）

### 2. 认证流程

Mediary 支持两种认证方式：

**方式A：API Key 认证（推荐，用于 AI Agent）**

在 `~/.hermes/skills/mediary/.env` 中配置：
```bash
MEDIARY_BASE_URL=https://your-domain.com/api/v1
MEDIARY_API_KEY=your_api_key_here
```

API Key 从前端 TOTP 管理页 → AI 接口密钥 生成。使用时 Header 为：
```
Authorization: Bearer MEDIARY_API_KEY
```

**方式B：JWT Token 认证（交互式登录）**

获取 Token 的流程：

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

> 下面用 `BASE_URL` 代替用户的实际后端地址（如 `https://your-domain.com/api/v1`）。

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

返回包含 `content` 字段（纯文本内容，仅供备份/搜索，前端不读此字段）。要获取前端显示的内容，用 `GET /blocks/document/:id`。

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

**🚨 关键：创建文档后必须写入 blocks，前端才能显示内容！**

前端渲染依赖 blocks 数据，不读 content 字段。正确流程：
1. `POST /documents` 创建文档（拿到 doc_id）
2. `PUT /blocks/document/:doc_id` 写入 blocks 内容

如果只写 content 不写 blocks，前端会显示空白。详见下方「块级编辑」章节。

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

**⚠️ 此接口只更新 content 字段，不影响 blocks。** 如果要更新前端显示的内容，必须用下方的「块级编辑」API 写入 blocks。

#### Patch 更新文档（部分字段）

**⚠️ 已知问题：** PATCH 端点可能返回 `Field validation for 'Patch' failed on the 'required' tag` 错误。如果遇到此问题，改用 PUT 全量更新（传入 title + doc_type + source 等完整字段）即可绕过。

```
PATCH $MEDIARY_BASE_URL/documents/:id/patch
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

> **🚨 核心概念：前端渲染依赖 blocks，不读 content 字段。**
> 无论创建还是更新文档内容，都必须通过 blocks API 写入，前端才能显示。
> `content` 字段仅作为备份/搜索用途，不影响前端渲染。

#### GetByDocID 获取文档的所有块
```
GET BASE_URL/blocks/document/:doc_id
```

#### ReplaceAll 替换文档的所有块
```
PUT BASE_URL/blocks/document/:doc_id
Body: [
  {"block_type": "paragraph", "content": "第一段"},
  {"block_type": "heading", "content": "标题", "level": 1}
]
```

**block_type 枚举：**
- `heading` — 标题（content 里必须带 `#`/`##`/`###` 等 markdown 标记）
- `paragraph` — 段落（content 里可用 `**加粗**`、`*斜体*` 等 markdown 语法）
- `list_item` — 列表项
- `code` — 代码块
- `quote` — 引用
- `divider` — 分割线

**🚨 content 必须用 markdown 语法写格式！**

前端渲染器解析 content 里的 markdown，不是靠 block_type + level 字段。
- 标题：`"content": "## 标题文本"` ✅  `"content": "标题文本"` ❌（会被渲染成普通正文）
- 加粗：`"content": "这是**重点**内容"` ✅
- 列表：block_type=list_item 时，content 直接写列表项文本即可

#### 完整写入流程（创建文档 + 写入 blocks）

```python
import json, urllib.request

BASE = "BASE_URL"
TOKEN = "YOUR_TOKEN"
headers = {"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}

# Step 1: 创建文档
doc_body = {"title": "标题", "content": "", "doc_type": "note", "source": "local", "tags": ["标签"]}
req = urllib.request.Request(f"{BASE}/documents", json.dumps(doc_body).encode(), headers=headers, method="POST")
doc_id = json.loads(urllib.request.urlopen(req).read())['data']['id']

# Step 2: 写入 blocks（这才是前端显示的内容）
# ⚠️ content 里必须用 markdown 语法写格式！
blocks = [
    {"block_type": "heading", "content": "## 标题文本", "level": 2},  # ## 不能省
    {"block_type": "paragraph", "content": "这是**加粗**的正文内容"},
    {"block_type": "list_item", "content": "列表项一"},
]
req = urllib.request.Request(f"{BASE}/blocks/document/{doc_id}", json.dumps(blocks).encode(), headers=headers, method="PUT")
urllib.request.urlopen(req)
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
Body: {
  "tomato_date": "2024-01-01",   // required, format YYYY-MM-DD
  "slot_time": "10:00",           // required, format HH:MM or HH:MM:SS
  "status": "done",               // optional, default "done"
  "task": "任务描述",              // optional
  "trace": "关联文档描述",         // optional
  "rating": 5,                    // optional, 1-5
  "duration": 30                  // optional, default 25 (minutes)
}
```

**⚠️ Upsert 用 `tomato_date` + `slot_time` 做唯一键**：同一天同一时间段的记录会被更新，不会重复创建。

**⚠️ `todo_id` 字段不在 Upsert 请求体中**：无法通过 Upsert 接口设置 todo 关联。如需在番茄记录中引用 todo，使用文本格式 `[[td:58]]`（短格式，易书写），前端 `RichText` 组件会自动渲染为可点击链接。

**🚨 Upsert 必须传完整的 `tomato_date`！** 如果省略 `tomato_date`，Go 的 `time.Parse` 会解析为零值 `0001-01-01`，导致记录存入错误日期，查询时看不到。POST 创建时必须传 `"tomato_date": "2026-05-29"`。**这是实际踩过的坑**——PUT 更新时不传 `tomato_date` 会导致记录"消失"（存到 `0001-01-01` 了）。

**📌 番茄引用最佳实践：** `task` 字段放简要概括，`trace` 字段放 `[[td:ID]]` 引用链接。trace 区域通过 `<RichText>` 渲染为可点击链接，点击打开 Todo 侧边栏。**不要在同一文本上同时渲染 `<TraceLinks>` 和 `<RichText>`**，会导致内容重复显示。

**前端渲染注意事项：**
- `task` 字段：用 `<RichText text={tomato.task} />` 渲染（支持 `[[td:ID]]` 链接）
- `trace` 字段：也用 `<RichText text={tomato.trace} />` 渲染
- 旧的 `todo_id` 关联显示已移除，统一用 `[[td:ID]]` 格式
- Tomato 页面不再需要查询 todo 列表（`todoApi.list()` 已移除）

#### Update 更新番茄钟记录
```
PUT BASE_URL/tomatoes/:id
Body: {"task": "新任务描述", "status": "done", "slot_time": "10:00:00", "duration": 30}
```

**⚠️ PUT 必须传 `status` 字段！** 后端 `status` 是 enum 类型，省略会导致 `Data truncated for column 'status'` 错误。建议 PUT 时传入完整字段（task + status + slot_time + duration）。**这是实际踩过的坑**——只传 task 不传 status 会导致更新失败。

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

N 由用户指定，默认 10。用户可以在对话中说"读前5个block"或"看前20条需求"来调整。

```bash
# N=5 示例：只看最新的 5 个 block
N=5
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  "BASE_URL/blocks/document/8387" | python3 -c "
import sys, json
blocks = json.load(sys.stdin).get('data', [])
for b in blocks[:$N]:
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
| mediary-dev-plan | 8387 | Mediary 开发需求计划（含prompt+done prompt合并，从新到旧） | 前 10 个（标题+最新需求） |

> 用户可通过对话告诉 Agent 新增/修改书签。Agent 应在 SKILL.md 的书签表中维护。

## ⚠️ 部署铁律：Dev 优先

**所有新功能必须先在 mediary-dev/ 开发并在 your-dev-domain.com 验证，确认 OK 后才能部署到生产 your-domain.com。**

绝对不允许跳过 dev 验证直接部署到生产。流程：
1. 在 `mediary-dev/` 写代码 → `go build` / `npm run build`
2. 重启 mediary-dev.service（端口 9000）或用 vite dev 验证
3. 用户在 your-dev-domain.com 确认 OK
4. 停 mediary.service → 用 cp 覆盖二进制和 dist 目录 → 启 mediary.service
5. 生产 your-domain.com 生效

部署产物步骤（dev→生产，不合并分支）：在 mediary-dev/ 构建后，将 backend/mediary-server 复制到 mediary/backend/，将 frontend/dist/ 内容复制到 mediary/frontend/dist/（先清空旧 dist），然后重启 mediary.service。systemd 需 `PORT=8081` 环境变量（dev 默认 9000）。.env 各目录独立不覆盖。

## Frontend Mobile-First Strategy (Phase 12)

**核心约束：** 所有前端改动必须同时适配移动端和桌面端，不能破坏任何一端。

- **Tailwind 断点策略**：mobile-first，基础样式 = 移动端，`md:` 前缀 = 桌面端（≥768px）
- **底部 Tab Bar**：移动端固定底部导航（首页/文档/Todo/番茄/我的），桌面端隐藏
- **侧边栏**：移动端收起为汉堡菜单或底部 Sheet，桌面端保持现有双栏
- **弹窗**：移动端改为全屏页面或底部 Sheet，桌面端保持 Modal
- **触摸区域**：所有可点击元素 ≥ 44px
- **安全区适配**：使用 `env(safe-area-inset-*)` 适配刘海屏/底部横条
- **PWA**：manifest.json + Service Worker，支持添加到主屏幕
- **需求文档**：[[8387]] Phase 12 区域（前 13 个 block）

## Frontend Component Architecture (Todo)

### Task Reference System `[[td:ID]]`

文本中写 `[[td:58]]`（短格式，易书写）会自动渲染为紫色可点击链接，点击后打开 Todo 侧边栏。

- **TaskRef.tsx** (`components/TaskRef.tsx`) — 解析 `[[td:ID]]` 并渲染为链接
  - `<RichText text="完成了 [[td:58]]" />` — 渲染文本+链接
  - `<TaskRefLink todoId={58} />` — 单个链接组件
  - `hasTaskRef(text)` / `extractTaskIds(text)` — 工具函数
- **TodoSidePanel.tsx** (`components/TodoSidePanel.tsx`) — 复用 `TodoFormContent` 的侧边栏
- **TodoFormContent** (`modules/todo/TodoForm.tsx` 的具名导出) — 从 TodoForm 抽取的纯表单内容，供 TodoForm（模态弹窗）和 TodoSidePanel（侧边栏）共用。修改 TodoFormContent 自动同步两处。

### PanelContext 扩展

`PanelContext` 同时支持文档侧边栏和 Todo 侧边栏，互斥（打开一个会关闭另一个）：

```typescript
// 文档侧边栏
openPanel(docId: number | "new")  // 打开文档面板
closePanel()                       // 关闭文档面板

// Todo 侧边栏
openTodoPanel(todoId: number)      // 打开 Todo 面板
closeTodoPanel()                   // 关闭 Todo 面板
```

Layout 中同时注册了 `DocumentSidePanel` 和 `TodoSidePanel`，全局可用。

### Tomato 页面渲染

番茄钟的 `task` 字段通过 `<RichText text={tomato.task} />` 渲染，trace 字段也通过 `<RichText>` 渲染，均支持 `[[td:ID]]` 链接。旧的 `todo_id` 显示已移除，统一用 `[[td:ID]]` 格式。

## Pitfalls

### [[id]] 文档链接导航会触发 auto-save 竞态
- 点击 `[[id]]` 链接后，`navigate(/documents/${docId})` 改变 URL
- `useParams` 立即更新 ID，但 `useQuery` 还在加载新文档
- **旧的 form/blocks 状态还在**，auto-save 检测到"变化"就用旧内容 PUT 到新文档 ID
- 如果目标文档不存在 → 500 错误（blocks API 无法为不存在的文档写入 blocks）
- **修复**：`DocumentEditor` 中用 `isNavigatingRef` + `prevIdRef` 检测 ID 变化，立即清空状态并阻断 auto-save
- 详见 `design-patterns` skill → 导航竞态保护模式

### [[id]] 文档链接有两个渲染路径
- **MarkdownRenderer**（预览模式）：`convertDocLinks()` 已内置 `[[id]]` → 可点击链接
- **BlockEditor**（编辑模式）：`renderBlockPreview()` 直接输出纯文本，不走 MarkdownRenderer。需要单独添加 `renderContentWithLinks()` 函数解析 `[[id]]`，并通过 `CustomEvent("navigate-doc")` 通知 DocumentEdit 页面跳转
- 两个路径完全独立，修改一个不影响另一个

### Tomato Upsert 缺少 tomato_date 会静默创建错误记录
- PUT `/api/v1/tomatoes` 的 upsertReq 要求 `tomato_date` 字段（binding:"required"）
- 如果传了空值或缺字段，Go 的 `time.Parse("2006-01-02", "")` 会返回 `0001-01-01`
- 记录会以零日期入库，查询正常日期时找不到 → 看起来像"记录消失"
- **正确做法**：每次 PUT 必须传完整的 `tomato_date`、`slot_time`、`status`

### PanelContext 扩展新面板类型
- 现有 PanelContext 支持 document 面板（`panelDocId` / `openPanel`）
- 扩展 todo 面板：新增 `panelTodoId` / `openTodoPanel` / `closeTodoPanel`
- 打开一个面板时关闭另一个（互斥逻辑）
- TodoSidePanel 复用 TodoFormContent（从 TodoForm 抽取的表单内容组件），保持功能同步更新

### Tailwind 动态类名陷阱
- 模板字符串如 `` bg-${color}-500/20 `` 在构建时被 Tailwind purge 静默删除
- 必须用完整类名或对象映射：`{ blue: "bg-blue-500/20", amber: "bg-amber-500/20" }`

## 微信文章收录（WeChat Article Collector）

# 微信文章收录器

## Purpose

当用户发送微信公众号文章链接（`mp.weixin.qq.com`）时，自动获取文章正文并收录到 Mediary 日记系统。支持图片转存到 GitHub 图床，避免链接失效。

## When To Use

- 用户消息以 `sl:` 开头（收录快捷命令）
- 用户在飞书/微信发一个微信文章链接
- 用户说"收录这篇文章"或"保存到日记"
- 检测到 `mp.weixin.qq.com/s/` 格式的链接

## 快捷命令格式

```
sl:https://mp.weixin.qq.com/s/xxxxx
sl:自定义标题[https://mp.weixin.qq.com/s/xxxxx]
sl:自定义标题(https://mp.weixin.qq.com/s/xxxxx)
```

解析规则：
1. 去掉 `sl:` 前缀
2. 如果有 `[标题]` 或 `标题[` 格式，提取自定义标题
3. 提取 URL（匹配 `https://mp.weixin.qq.com/s/` 开头）
4. 无自定义标题时，从页面 og:title 自动获取

## 配置要求


```bash
# Mediary API（必须）
MEDIARY_BASE_URL=https://your-dev-domain.com/api/v1
MEDIARY_API_KEY=your_api_key_here

# GitHub 图床（可选，不配置则不转存图片）
GITHUB_IMAGE_OWNER=your_github_username
GITHUB_IMAGE_REPO=YOUR_IMAGE_REPO
GITHUB_IMAGE_BRANCH=main
GITHUB_IMAGE_TOKEN=your_github_token_here
```

## Workflow

### Step 1: 获取文章内容

用 curl 获取微信文章页面（手机 UA 绕过反爬）：

```bash
curl -s -L "ARTICLE_URL" \
  -H "User-Agent: Mozilla/5.0 (Linux; Android 14; Pixel 8 Pro) AppleWebKit/537.36" \
  -H "Accept-Language: zh-CN,zh;q=0.9" > /tmp/wechat_article.html
```

### Step 2: 解析 HTML 并转为 Blocks

用 BeautifulSoup 解析，保留排版（标题、加粗、图片、引用等）：

```python
from bs4 import BeautifulSoup, NavigableString

def process_element(el):
    """递归提取元素，返回 block"""
    if el.name == 'img':
        src = el.get('data-src') or el.get('src', '')
        if src and 'mmbiz.qpic.cn' in src:
            return {"block_type": "paragraph", "content": f"![图片]({src})"}
    
    if el.name == 'p':
        parts = []
        for child in el.children:
            if child.name in ['strong', 'b']:
                parts.append(f"**{child.get_text(strip=True)}**")
            elif child.name == 'img':
                src = child.get('data-src') or child.get('src', '')
                if src:
                    parts.append(f"![图片]({src})")
            else:
                parts.append(child.get_text(strip=True))
        return {"block_type": "paragraph", "content": ' '.join(parts)}
    # ... 更多元素处理
```

### Step 3: 转存图片到 GitHub 图床（可选）

如果配置了 GitHub Token，自动转存图片：

```python
import hashlib, base64

def upload_to_github(img_url, config):
    """下载图片并上传到 GitHub，返回新 URL"""
    # 1. 检查缓存（避免重复上传）
    img_hash = hashlib.md5(img_data).hexdigest()[:12]
    cache_key = f"{img_hash}.{ext}"
    
    # 2. 检查 GitHub 是否已存在
    check_url = f"https://api.github.com/repos/{owner}/{repo}/contents/{path}"
    # 如果存在，直接返回现有 URL
    
    # 3. 不存在则上传
    content_base64 = base64.b64encode(img_data).decode()
    # PUT /repos/{owner}/{repo}/contents/{path}
    
    # 4. 返回新 URL
    return f"https://raw.githubusercontent.com/{owner}/{repo}/{branch}/{path}"
```

**去重逻辑：**
- 用图片内容的 MD5 哈希作为文件名
- 上传前先检查 GitHub 是否已存在该文件
- 存在则跳过上传，直接返回 URL

### Step 4: 写入 Mediary

创建文档并写入 blocks：

```python
# 创建文档
doc_body = {
    "title": title,
    "content": "微信文章收录",
    "doc_type": "note",
    "source": "local",
    "tags": ["微信文章"]
}
# POST /documents

# 写入 blocks（前端才能显示）
blocks = [
    {"block_type": "heading", "content": f"# {title}", "level": 1},
    {"block_type": "paragraph", "content": f"> **来源：** 微信公众号 · {author}"},
    {"block_type": "paragraph", "content": f"> **原文链接：** [{url}]({url})"},
    {"block_type": "divider", "content": "---"},
    # ... 正文 blocks
]
# PUT /blocks/document/{doc_id}
```

### Step 5: 返回结果

告诉用户：
- 文章标题
- Mediary 文档 ID 和访问链接
- 图片转存数量（如有）

## 完整收录流程

```python
import urllib.request, json, hashlib, base64, re
from bs4 import BeautifulSoup, NavigableString
from datetime import datetime

# 1. 读取配置
env = load_env('~/.hermes/skills/mediary/.env')

# 2. 抓取文章
html = fetch_article(url)

# 3. 解析 HTML
title, author, blocks = parse_html(html)

# 4. 转存图片（如果配置了 GitHub）
if env.get('GITHUB_IMAGE_TOKEN'):
    blocks = upload_images(blocks, env)

# 5. 创建 Mediary 文档
doc_id = create_mediary_doc(title, blocks, env)

# 6. 返回结果
print(f"✅ 收录成功！文档 ID: {doc_id}")
```

## Pitfalls

1. **微信反爬**：curl + 手机 UA（Android Pixel 8）通常可以绕过
2. **图片 data-src**：微信图片用 `data-src` 属性（非 `src`），需同时匹配
3. **重复上传**：用 MD5 哈希检查，避免重复上传同一图片
4. **GitHub 422 错误**：文件已存在时会返回 422，需要先检查
5. **图片格式**：根据 Content-Type 确定扩展名（png/jpg/gif/webp）
6. **blocks 必须写入**：只写 content 不写 blocks，前端显示空白

## 使用示例

```
用户: sl:https://mp.weixin.qq.com/s/ayrD-PlL4d47lrgtaS7ZcQ

助手执行:
1. curl 获取文章 HTML
2. BeautifulSoup 解析标题、作者、正文
3. 提取图片链接，转存到 GitHub（12张）
4. 创建 Mediary 文档，写入 blocks
5. 返回: "✅ 收录成功！《标题》，文档 ID: 8543，转存 12 张图片"
```


## Related Skills


## Limitations

- Token 对所有端点有效，暂无细粒度权限控制
- 文档内容无版本管理，覆盖更新需自行备份
- 搜索为模糊匹配，无全文索引
- `source` 字段只能是 `notion`/`feishu`/`local`，不支持自定义值
