# Mediary Skill 问题手册

## 2026-06-10 - .env 文件被错误覆盖事故

### 问题描述

维护 mediary skill 时，错误地将本地 `.env` 文件覆盖为占位符，导致真实 API Key 丢失。

### 根因

**对 git stash 行为理解错误：**

`.env` 从创建第一天起就被 `.gitignore` 保护，从未进入 git 追踪范围。`git stash` 只能保存**已追踪文件**的改动，对 untracked 文件无效。

错误的操作链：
1. 执行 `git add SKILL.md` 等文件（commit 204027c）
2. 此时 `.env` 是 **untracked** 状态
3. 执行 `git stash` → `.env` 实际上**未被保存**（因为 untracked）
4. 执行 `git pull --rebase origin main`
5. rebase 过程中工作区 `.env` 被覆盖为远程状态
6. 执行 `git stash pop` → **什么都没恢复**（stash 里根本没有 .env）
7. 误以为 `.env` 丢失，错误地重建为占位符 → **真实 Key 被覆盖**

### 教训

1. **git stash 不保存 untracked 文件**。对于 `.env` 这种 always-untracked 文件，stash 毫无保护作用
2. **对受 .gitignore 保护的文件，本地备份才是唯一保护手段**，不是 git
3. **恐慌导致过度反应**：发现问题后没有冷静分析 git 对象状态，就急于"修复"
4. **正确做法**：`.env` 类文件丢失时，应从可信来源（用户、密码管理器）重新写入，而非试图从 git 恢复

### 预防

- 重要配置文件（.env、credentials）应在本地有备份副本
- git 操作前先 `git status` 确认工作区状态，不要基于假设行动
- 对于 untracked 的敏感文件，操作 git 前先确认它是否真的被 stash/git 对象保存

### 恢复

需要用户重新提供 API Key，写入 `/root/.hermes/skills/mediary/.env`

---

## 当前已知问题

- 2026-06-10：`.env` 文件 API Key 需要用户重新提供（见上方）
