# Mediary-Dev 部署与开发陷阱

## Nginx 扩展配置陷阱

your-dev-domain.com 的 nginx 主配置在 `/www/server/panel/vhost/nginx/your-dev-domain.com.conf`，但关键 location 块在**扩展目录**中：

扩展目录: `/www/server/panel/vhost/nginx/extension/your-dev-domain.com/`
- `api-proxy.conf` — /api/ 反代到后端端口
- `site_total.conf` — 访问统计
- `spa-router.conf` — SPA 路由 fallback

**⚠️ 不要在主配置文件中重复添加 /api/ location！** 会导致 nginx `duplicate location` 报错。修改 API 反代时必须改 `api-proxy.conf`。

## Vite 代理配置

`frontend/vite.config.ts` 的 proxy target 必须与后端端口一致：
- dev 后端: localhost:9000（不是 8080！）
- 生产后端: localhost:8081

## BlockEditor vs MarkdownRenderer 渲染路径（Phase 1 渲染引擎升级后）

文档有**两条完全独立的渲染路径**，修改渲染逻辑时必须同时检查！

### 路径 1: MarkdownRenderer（非块文档、共享文档页）
- 文件：`components/MarkdownRenderer.tsx`
- 使用 `renderMarkdownSync()` 从 `lib/md-renderer` 渲染完整 Markdown
- 支持：代码高亮、`[[id]]` 链接、checkbox 交互

### 路径 2: BlockEditor（块文档编辑/预览）
- 文件：`components/BlockEditor.tsx`
- 有自己的 `renderBlockPreview()` 函数
- 使用 `renderInlineMarkdown()` 渲染每个 block 的内联 Markdown（粗体、斜体、行内代码、`[[id]]` 等）
- 实现：调用 `renderMarkdownSync(text)` → 去掉外层 `<p>` → `<span dangerouslySetInnerHTML>`

### 渲染引擎架构（`lib/md-renderer/`）
- `index.ts` — 主入口，`renderMarkdownSync()` / `renderMarkdown()`
- `renderer.ts` — marked renderer 配置（h1-h6/p/blockquote/code/table/list/...）
- `extensions/registry.ts` — 插件注册中心（`registerExtension()`）
- `extensions/highlight.ts` — 代码高亮（highlight.js，20种语言）
- `extensions/doc-links.ts` — `[[id]]` 文档链接
- `themes/` — CSS 主题系统（base.css + default.css + CSS 变量）

导航通过 `CustomEvent("navigate-doc")` + `useEffect` 监听实现。

## marked.js v18 集成陷阱

### 陷阱 1: Renderer 类型不存在
- ❌ `import type { MarkedRenderer } from "marked"` — 不存在
- ✅ `import type { RendererApi } from "marked"` — 正确类型名
- 或者直接用 `Record<string, any>` 作为返回类型（最省事）

### 陷阱 2: renderer 方法的 this 上下文
marked v18 的 renderer 方法通过 `this.parser` 访问解析器，TypeScript 无法推导：
```typescript
// ❌ TypeScript 报错：Property 'parser' does not exist
heading({ tokens, depth }) {
  const text = this.parser.parseInline(tokens);
}
// ✅ 显式声明 this 类型
heading(this: any, { tokens, depth }) {
  const text = this.parser.parseInline(tokens);
}
```

### 陷阱 3: 扩展 renderer 必须与基础 renderer 合并
```typescript
// ❌ 错误：扩展的 renderer 被忽略
marked.use({ renderer: baseRenderer, ...mergeExtensions(extensions) });
// ✅ 正确：先合并所有 renderer
const mergedRenderer = { ...baseRenderer };
for (const ext of extensions) {
  if (ext.renderer) Object.assign(mergedRenderer, ext.renderer);
}
marked.use({ renderer: mergedRenderer, ...mergeExtensions(extensions) });
```
**症状**：加粗/斜体不渲染 — 因为 highlight 扩展的 `code` 方法覆盖了基础 renderer 的所有方法。

### 陷阱 4: DOMPurify SVG 白名单
Mermaid/KaTeX 输出的 SVG 需要额外白名单：
```typescript
DOMPurify.sanitize(html, {
  ADD_TAGS: ["svg", "circle", "path", "line", "polyline", "polygon", "g", "text", "tspan"],
  ADD_ATTR: ["viewBox", "fill", "stroke", "stroke-width", "cx", "cy", "r", "d", "points"],
});
```

## Tomato API 陷阱

- `PUT /api/v1/tomatoes/:id` 必须包含 `tomato_date` 和 `slot_time` 字段
- 缺少 `tomato_date` 时日期变成零值，查询时找不到记录（看起来像"消失"）
- Upsert 按 tomato_date + slot_time 匹配，同时间段会互相覆盖

## 组件复用模式：TodoFormContent

`TodoForm.tsx` 导出两个：
- `TodoFormContent` (named export): 纯表单内容，无 modal 包装，供 TodoSidePanel 复用
- `default TodoForm`: 带 modal 包装的完整弹窗

新增侧边栏时复用 TodoFormContent，自动获得表单功能更新。
