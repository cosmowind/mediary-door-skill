# Mediary Skill for Hermes Agent

让 AI Agent 通过 API Key 自主读写 Mediary 日记文档库的操作手册。

## 安装

将本仓库克隆到 Hermes Agent 的 skills 目录：

```bash
cd ~/.hermes/skills/
git clone https://github.com/cosmowind/hermes-skill-mediary.git mediary
```

## 配置

1. 登录你的 Mediary Web UI
2. 进入「设备」页面 →「AI 接口密钥」tab
3. 点击「生成 API Key」，复制保存
4. 在 AI Agent 中使用该 Key 调用 API

## 使用

安装后，Hermes Agent 会自动加载此 Skill。当对话涉及文档读写、日记管理时，Agent 会参考此手册调用 Mediary API。

### 支持的操作

- 创建/读取/更新/删除文档
- 搜索文档（关键词+标签+类型）
- 标签管理（列出/创建/删除/批量操作）
- 按日期范围筛选

## 要求

- Mediary 后端已部署并运行
- 用户已生成 API Key
- `curl` 或任意 HTTP 客户端
