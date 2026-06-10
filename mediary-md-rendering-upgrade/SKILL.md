---
name: mediary-md-rendering-upgrade
description: Mediary Markdown 渲染引擎升级 — 用 doocs/md 架构替换 react-markdown。插件式可成长框架，Phase 1-3 渐进式开发。
triggers:
  - MD渲染
  - markdown渲染
  - md-renderer
  - 文档排版
---

# Mediary MD 渲染引擎升级

## 概述
用 doocs/md (kgipsonmymail/md, ~/md) 的渲染架构替换 Mediary 当前简陋的 react-markdown 方案。
插件式可成长框架，每个 Phase 结束都可部署使用。

## 项目状态 (2026-06-10)
- **Phase 1 已部署** — 核心引擎 + 代码高亮 + 主题，dev 分支运行正常
- **Bug 1 (已修):** `mergeMarkedExtensions` renderer 丢失，已手动合并 renderer
- **Bug 2 (已修):** BlockEditor 独立渲染，已替换为 `renderInlineMarkdown`
- **Bug 3 (已修):** 列表渲染 `[object Object]` — marked v18 list items 是 token 对象，需调用 `this.listitem(item)` 而非 `.join("")`
- **Bug 4 (已修):** 表格/列表 CSS 选择器不匹配 — 选择器用了 `.md-rendered table`/`.md-rendered th` 但渲染器输出的是 `<div class="md-table-wrapper"><table class="md-table"><th class="md-th">`，中间隔了 wrapper 层且 th 的 class 是 `md-th` 不是 `md-rendered`。已修正为 `.md-rendered .md-table`、`.md-rendered .md-th`、`.md-rendered .md-td`。主题 CSS（JS 注入）已正确定义了 `.md-table`/`.md-th`/`.md-td` 样式，静态 CSS 只是补充 override。
- **Bug 5 (已修):** alert/ruby/toc 扩展双层嵌套导致 renderer 未注册 — `getAllMarkedExtensions()` 一次 flatMap 不足，需两次；且 sub-extensions 的 renderer 不能进 `topLevelRenderers`（白名单验证路径），必须只走 `allInnerExtensions`。
- **Bug 6 (已修):** `blockToWechatHtml` 两套渲染路径导致 shared 页面加粗主题色不一致 — paragraph/list/alert/blockquote 用 `renderInlineMarkdown`（inline style ✅），但 default case 和 table cells 用 `md()` → `renderMarkdownSync`（输出 `<strong class="md-strong">` ❌）。shared 页面没有引入任何 CSS 文件，所以 `.md-strong` 无颜色定义，始终显示黑色。reading view 通过 `injectThemeCSS` 在 `:root` 注入 `.md-strong { color: ${color} !important }` 所以正常。**修复**：default case 和 table cells/th 全部改用 `renderInlineMarkdown(content, color)`，统一走 inline style 路径。删除废弃的 `md()` 函数和 `renderMarkdownSync` import。

## CSS 调试方法论

**核心原则**：写 CSS 前必须先确认实际 HTML 输出结构。

1. **查渲染器源码** `renderer.ts` — 看 `createRenderer()` 返回的 HTML 标签和 class 名
2. **查主题 CSS** `themes/default.css` — 已定义 `.md-table`、`.md-th`、`.md-td` 等类的样式
3. **查编译后 JS** `dist/assets/index-*.js` — 主题 CSS 通过 `injectThemeCSS()` 注入，搜 `$M` 函数确认存在
4. **确认选择器匹配** — 静态 CSS 的选择器必须和渲染器输出的 class 名完全对应
5. **双重样式冲突** — 同时存在主题 CSS（JS 注入）和静态 CSS（index.css）时，注意 specificity 优先级

**本项目渲染器输出结构**（以表格为例）：
```html
<div class="md-rendered">
  <div class="md-table-wrapper">   <!-- ← wrapper 层 -->
    <table class="md-table">       <!-- ← 表本身 -->
      <thead><tr><th class="md-th">header</th></tr></thead>
      <tbody><tr><td class="md-td">cell</td></tr></tbody>
    </table>
  </div>
</div>
```
因此正确的 CSS 选择器是 `.md-rendered .md-table`、`.md-rendered .md-th`、`.md-rendered .md-td`，而不是 `.md-rendered table`、`.md-rendered th`。

## 架构设计

### 文件结构
```
frontend/src/lib/md-renderer/
├── types.ts              ← MdExtension 接口 + MdRendererConfig
├── utils.ts              ← escapeHtml, simpleHash
├── renderer.ts           ← marked renderer (h/p/blockquote/code/table/list/...)
├── index.ts              ← 主入口 + 管线串联
├── extensions/
│   ├── registry.ts       ← 插件注册中心
│   ├── highlight.ts      ← 代码高亮 (Phase 1)
│   └── doc-links.ts      ← [[id]] 文档链接 (Phase 1)
└── themes/
    ├── base.css          ← 基础排版
    ├── default.css       ← 经典主题
    └── index.ts          ← 主题管理器 + CSS 注入
```

### 插件接口
```typescript
interface MdExtension {
  name: string;
  markedExtensions?: MarkedExtension[];
  css?: string;
  postProcess?: (html: string) => Promise<string>;
}
```

### 注册方式
```typescript
import { registerExtension, highlightExt, docLinksExt } from './extensions';
registerExtension(highlightExt);
registerExtension(docLinksExt);
```

## 已安装依赖
- `marked` 18.0.4
- `highlight.js` 11.11.1
- `isomorphic-dompurify` 3.15.0

## 关键组件修改
- `MarkdownRenderer.tsx` — 替换 react-markdown 为新引擎
- `SharedDocument.tsx` — 替换直接使用的 ReactMarkdown
- `BlockEditor.tsx` — 替换 `renderContentWithLinks` 为 `renderInlineMarkdown`

## 三个 Phase

### Phase 1 (当前)
- 核心渲染引擎 (marked + DOMPurify)
- 代码高亮 (highlight.js, 20种语言)
- [[id]] 文档链接
- CSS 主题系统 (经典/优雅/简洁)

### Phase 2
- 数学公式 (KaTeX/MathJax)
- Mermaid 图表

### Phase 3
- GFM Alerts + Obsidian Callouts (20+ 种提示框)
- 脚注 + 标记语法 (==高亮==, ++下划线++)
- Ruby 注音 + TOC 目录

## 已知陷阱
1. **BlockEditor 独立渲染** — BlockEditor 有 `renderContentWithLinks` 不经过 MarkdownRenderer，必须同时替换
2. **mergeMarkedExtensions renderer 丢失** — marked.use() 的 extensions 数组中 renderer 属性被跳过，需手动合并 renderer
3. **Vite build .user.ini** — 构建到 dist_new 再替换，不要直接 build 到 dist
4. **CSS 作用域隔离** — 容器类名是 `md-rendered`（不是 `prose`），所有自定义 CSS 选择器必须用 `.md-rendered` 前缀。Tailwind preflight 会重置 `table`/`th`/`td` 的 border 为 0，`ol`/`ul` 的 list-style 为 none。
5. **静态 CSS 优先原则（重要）** — markdown 样式必须直接写在 `frontend/src/index.css` 的静态 CSS 里，不要依赖 JS 动态注入。主题 CSS（`themes/default.css`）通过 `injectThemeCSS()` 在 `useEffect` 中注入，但 `useEffect` 在 hydration 之后才执行，导致首次渲染时样式缺失（flash of unstyled content），且 12 小时 Nginx 缓存使用户可能长期看不到更新。正确做法：在 `index.css` 中用 `.md-rendered .md-[className]` 选择器覆盖 Tailwind preflight 重置，例如 `.md-rendered .md-ol{list-style-type:decimal}`、`.md-rendered .md-table{border-collapse:collapse}`、`.md-rendered .md-th{border:1px solid #d1d5db}` 等。改完 CSS 后必须重新 `npm run build`。
6. **Block 数据模型不变** — 只升级渲染层，后端 API 和存储格式不受影响
7. **marked v18 sub-extension 双层嵌套** — alert/ruby/toc 等扩展用 `markedExtensions: [{ extensions: [...] }]` 结构，registry.ts 的 `getAllMarkedExtensions()` 必须两次 flatMap 才能展开到底层 tokenizer/renderer。一次 flatMap 只拿到外层 wrapper `{ name, markedExtensions: [...] }`，无法注册到 marked。
8. **marked v18 renderer 白名单验证** — `initRenderer()` 中，有 `ext.renderer` 但无 `ext.extensions` 的 sub-extension（如 alert/ruby/toc），其 renderer 方法名（如 `ruby`/`toc`/`alert`）不在 KNOWN_TOKENS 白名单中，触发 "renderer 'X' does not exist"。正确做法：sub-extensions 只进 `allInnerExtensions`（通过 extensions 数组注册），**不**进 `topLevelRenderers`（白名单验证路径）。
9. **SharedDocument 颜色语义（重要）** — blockquote/H2(code header)/H4+ heading 用主题色 `${color}`；paragraph/list/default/table 等正文内容用 `renderInlineMarkdown(content, color)` 输出 inline style（color 固定 `#374151` 由 inline style 保证）；H1/H3 文字固定 `#1f2937`（与装饰元素分离）。不要把 paragraph/list 的 color 也改成主题色，会导致正文全部变成主题色。**重要**：`blockToWechatHtml` 内部只允许 `renderInlineMarkdown` 一种渲染方式，禁止调用 `renderMarkdownSync`（CSS class 路径），因为 shared 页面没有 CSS 导入。
10. **Tailwind v4 构建时内联 CSS variable fallback（重要）** — `@tailwindcss/vite` 插件在构建时会将 `var(--md-primary-color, #6366f1)` 内联为 `#6366f1`，导致 CSS 文件里的 `var()` 在运行时已经变成硬编码色，无法通过 JS 改变 CSS variable 来更新主题色。症状：修改页面主题色后，`.md-strong` 等颜色不变。调试方法：`grep -o '.md-strong[^}]*}' dist/assets/*.css` 看构建产物是否包含 `var()`（正确）还是硬编码色（被内联）。
    **解决方案（reading view）**：颜色值必须在运行时通过 JS 模板字符串拼接进 `injectThemeCSS()` 注入的 `<style>` 标签中，不能依赖 CSS 文件里的 `var()` 表达式：
    ```ts
    const colorCSS = primaryColor
      ? `.md-rendered .md-strong { color: ${primaryColor} !important; }`
      : "";
    ```
    **空值陷阱**：`pagePrimaryColor` 读取页面 `getComputedStyle(document.documentElement).getPropertyValue('--primary-color')`，但页面本身没有设置 `--primary-color` 时返回空字符串 `""`，导致拼接出 `color:  !important`（空值），反而覆盖了 CSS 文件的 fallback 值 `#6366f1`，使加粗完全无颜色。必须加空值保护：`pagePrimaryColor || ""` 后判断是否为空，为空时 `colorCSS = ""` 不注入。

    **解决方案（shared/公众号版）**：shared 页面不走 `MarkdownRenderer` 的 CSS class 路径，而是用 `blockToWechatHtml` 手动拼接 HTML，绝不能依赖任何 CSS 导入。所有加粗必须用 `renderInlineMarkdown(text, color)` 输出 `style="color:${color} !important;font-weight:bold;"` inline style。禁止在 `blockToWechatHtml` 里调用 `renderMarkdownSync`（输出 CSS class），因为 shared 页面没有引入任何 CSS 文件，`.md-strong` 无颜色定义。

## 参考项目
- 源码: ~/md (kgipsonmymail/md, fork of doocs/md)
- 渲染核心: ~/md/packages/core/src/renderers/ 和 ~/md/packages/core/src/extensions/
- 主题样式: ~/md/packages/core/src/themes/

## 计划文件
- docs/plans/2026-06-03-md-rendering-upgrade.md
