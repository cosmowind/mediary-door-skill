# Mediary Skill 问题手册

## 当前已知问题

（暂无）

---

## 已归档条目

### .env 文件被错误覆盖事故

> ⚠️ 此条已迁移到 WCS `docs/error_book.md`（根因更详细）。此处仅保留项目特定恢复信息。

**问题**：维护时错误地将 `.env` 覆盖为占位符，API Key 丢失
**根因**：`git stash` 对 untracked 文件无效，rebase 导致工作区被覆盖
**恢复**：用户重新提供 API Key，写入 `/root/.hermes/skills/mediary/.env`
**记录时间**：2026-06-10

---

### CSS var() fallback 在变量未定义时不生效

> ⚠️ 此条已迁移到 WCS `docs/error_book.md`。此处仅保留项目特定信息。

**问题**：wechat view 中 block 颜色不跟随主题变化，刷新后好了
**根因**：CSS `var()` fallback 在变量未定义时无效，fallback 是默认值
**涉及文件**：wechat view / MarkdownRenderer / injectThemeCSS
**记录时间**：2026-06-10

---

### marked v18 renderer 白名单验证规则

> ⚠️ 此条已迁移到 WCS `docs/error_book.md`。此处仅保留项目特定信息。

**问题**：`renderer 'alert' does not exist`
**根因**：marked v18 KNOWN_TOKENS 白名单，自定义 renderer 未注册
**涉及文件**：`src/renderer/index.ts` 的 `KNOWN_TOKENS`
**记录时间**：2026-06-10
