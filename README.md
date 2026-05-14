# Mediary Door Skill

Hermes Agent 技能：让 AI 通过 API Key 自主读写 Mediary 日记文档库。

## 安装

```bash
cd ~/.hermes/skills/
git clone https://github.com/cosmowind/mediary-door-skill.git mediary
```

## 前置条件

1. Mediary 后端已部署并运行
2. 你有一个 Mediary 账号

## 配置 API Key

1. 登录 Mediary Web UI
2. 进入「设备」页面 →「AI 接口密钥」tab
3. 点击「生成 API Key」
4. **立即复制保存**（明文只显示一次）

## 使用

安装后，Hermes Agent 会自动加载此 Skill。当你让 AI 操作 Mediary 文档时，AI 会：

1. 询问你的后端地址（如 `https://your-domain.com/api/v1`）
2. 使用你的 API Key 调用 API
3. 完成文档读写、搜索、标签管理等操作

### 支持的操作

- 📝 创建/读取/更新/删除文档
- 🔍 搜索文档（关键词 + 标签 + 类型 + 日期范围）
- 🏷️ 标签管理（列出/创建/删除/批量操作）
- 📋 获取文档列表（分页）

## 安全说明

- API Key 通过 Bearer Token 传输，后端 bcrypt 哈希存储
- 每个用户独立的 Key，可随时撤销重新生成
- 不要在公开场合泄露你的 API Key
