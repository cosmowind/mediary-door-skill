---
name: wechat-to-mediary
description: 微信文章收录到 Mediary 日记系统。当用户发送微信公众号文章链接时，自动获取正文、转存图片到 GitHub 图床并写入 Mediary。
tags: [mediary, wechat, content, diary, image-host]
---

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

在 `~/.hermes/skills/mediary/.env` 中配置：

```bash
# Mediary API（必须）
MEDIARY_BASE_URL=https://your-mediary-domain.com/api/v1
MEDIARY_API_KEY=your_api_key_here

# GitHub 图床（可选，不配置则不转存图片）
GITHUB_IMAGE_OWNER=your_github_username
GITHUB_IMAGE_REPO=your_image_repo
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
