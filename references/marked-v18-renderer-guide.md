# marked.js v18 Renderer 适配指南

## 问题

marked.js v18 的 TypeScript 类型系统与旧版本差异很大：
- `MarkedRenderer` 类型不存在，导出的是 `RendererApi`
- renderer 方法需要 `this.parser` 上下文，但 TypeScript 无法从 plain object 推导
- 扩展的 `renderer` 函数接收 `Tokens.Generic`，不能用自定义类型

## 解决方案

### renderer.ts 返回类型
```typescript
// ❌ 旧写法（编译报错）
import type { MarkedRenderer } from "marked";
export function createRenderer(): Partial<MarkedRenderer> { ... }

// ✅ 正确写法
export function createRenderer(): Record<string, any> {
  return {
    heading(this: any, { tokens, depth }: { tokens: any[]; depth: number }) {
      const text = this.parser.parseInline(tokens);
      return `<h${depth} class="md-h md-h${depth}">${text}</h${depth}>`;
    },
    // ... 所有 renderer 方法都加 this: any
  };
}
```

### 扩展 renderer 函数类型
```typescript
// ❌ 旧写法（类型不兼容）
renderer(token: { docId: number }) { ... }

// ✅ 正确写法
renderer(token: any) {
  return `<a href="/documents/${token.docId}">...</a>`;
}
```

### 扩展注册
```typescript
import { marked } from "marked";

// marked.use() 接受 renderer 和 extensions
marked.use({
  renderer: createRenderer(),  // Record<string, any> 兼容
  ...mergeMarkedExtensions(extensions),  // tokenizer + walkTokens
});
```

### DOMPurify SVG 白名单
```typescript
DOMPurify.sanitize(html, {
  ADD_TAGS: ["svg", "circle", "path", "line", "polyline", "polygon", "g", "text", "tspan"],
  ADD_ATTR: ["viewBox", "fill", "stroke", "stroke-width", "cx", "cy", "r", "d", "points"],
});
```

## Phase 2/3 扩展注册模板

```typescript
// extensions/xxx.ts
import type { MdExtension } from "../types";

export const xxxExt: MdExtension = {
  name: "xxx",
  markedExtensions: [{
    extensions: [{
      name: "xxxSyntax",
      level: "block" as const,
      start(src: string) { return src.match(/pattern/)?.index; },
      tokenizer(src: string) { /* return token */ },
      renderer(token: any) { return `<div>...</div>`; },
    }],
  }],
  css: `.md-xxx { ... }`,
};
```

然后在 `index.ts` 中：
```typescript
import { xxxExt } from "./extensions/xxx";
registerExtension(xxxExt);
```
