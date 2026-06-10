# GitHub 图床图片转存模式

## 用途
将外部图片（微信 CDN、临时链接等）下载后转存到 GitHub 仓库，生成永久链接。

## 配置（放入 .env，不要硬编码）
```bash
GITHUB_IMAGE_OWNER=your_github_username
GITHUB_IMAGE_REPO=your_image_repo
GITHUB_IMAGE_BRANCH=main
GITHUB_IMAGE_TOKEN=your_github_pat_here
```

## 核心流程
1. 下载外部图片（带 Referer header 绕过防盗链）
2. 计算 MD5 哈希作为文件名（天然去重）
3. 先检查 GitHub 是否已存在该文件（避免 422 错误）
4. 不存在则 base64 编码后 PUT 上传
5. 返回 raw.githubusercontent.com 永久链接

## Pitfalls
- 微信图片需要 `Referer: https://mp.weixin.qq.com/` header
- GitHub 文件已存在时返回 422，必须先 GET 检查
- 图片路径用 `images/年月/哈希.ext` 格式便于管理
- .gitignore 中排除 .env 文件，避免泄露 token
