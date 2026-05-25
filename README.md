# Mediary Skill for Hermes Agent

一份完整的 Mediary 日记后端 API 对接手册，使 AI Agent 能够通过 API Key 自主完成文档读写、标签管理、搜索过滤等任务。

## 这是什么？

这是一个 [Hermes Agent](https://github.com/cosmowind/hermes-agent) 的 Skill（技能），提供了 Mediary 日记系统的完整 API 参考和最佳实践。

**Mediary** 是一个轻量级的日记/文档管理系统，支持：
- 文档 CRUD（日记、笔记、专题）
- 块级编辑（Block-based editing）
- 标签管理
- Todo / 番茄钟 / 账单 / 目标管理
- AI Agent API Key 认证

## 快速开始

### 1. 安装

将此 skill 目录复制到 Hermes 的 skills 目录：

```bash
cp -r mediary ~/.hermes/skills/
```

### 2. 配置

复制 `.env.example` 为 `.env` 并填入你的配置：

```bash
cp .env.example .env
# 编辑 .env，填入你的 Mediary 地址和 API Key
```

### 3. 生成 API Key

1. 登录 Mediary 前端
2. 进入「设备管理」页面
3. 找到「AI 接口密钥」区块
4. 点击「生成」按钮
5. 复制生成的 Key 到 `.env` 文件

### 4. 使用

在 Hermes Agent 对话中，Agent 会自动加载此 skill 来操作 Mediary：

```
用户：帮我创建一篇日记，标题是「今日小记」
Agent：（自动调用 Mediary API 创建文档）
```

## 功能模块

| 模块 | 说明 |
|------|------|
| 文档 | CRUD、搜索、置顶、回收站、批量操作 |
| 块编辑 | Block-based 编辑，支持 Markdown 语法 |
| 标签 | 创建、删除、统计 |
| Todo | 任务管理、依赖关系 |
| 番茄钟 | 时间管理、统计、热力图 |
| 账单 | 记账、统计 |
| 目标 | 目标管理 |
| 账号 | 多平台账号管理 |
| 偏好 | 用户设置 |
| 总结 | AI 生成日记总结 |

## 环境变量

| 变量 | 必填 | 说明 |
|------|------|------|
| `MEDIARY_BASE_URL` | ✅ | Mediary 后端地址（含 `/api/v1`） |
| `MEDIARY_API_KEY` | ✅ | API Key（从前端生成） |
| `MEDIARY_BOOKMARK_PLAN` | ❌ | 书签：开发计划文档 ID |
| `MEDIARY_BOOKMARK_LOG` | ❌ | 书签：开发日志文档 ID |

## License

MIT
