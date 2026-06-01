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

## BlockEditor vs MarkdownRenderer 渲染路径

文档编辑页（DocumentEdit）使用 BlockEditor 组件渲染块内容，**不走 MarkdownRenderer**。

- `MarkdownRenderer`: 有 `convertDocLinks()` 将 `[[id]]` 转为可点击链接
- `BlockEditor.renderBlockPreview()`: 直接输出纯文本 `{content}`，不转换 `[[id]]`

要让 BlockEditor 支持自定义链接格式，需在 `renderBlockPreview` 函数前添加解析函数，然后在 heading/paragraph/list 渲染器中使用。导航通过 `CustomEvent("navigate-doc")` + `useEffect` 监听实现。

## Tomato API 陷阱

- `PUT /api/v1/tomatoes/:id` 必须包含 `tomato_date` 和 `slot_time` 字段
- 缺少 `tomato_date` 时日期变成零值，查询时找不到记录（看起来像"消失"）
- Upsert 按 tomato_date + slot_time 匹配，同时间段会互相覆盖

## 组件复用模式：TodoFormContent

`TodoForm.tsx` 导出两个：
- `TodoFormContent` (named export): 纯表单内容，无 modal 包装，供 TodoSidePanel 复用
- `default TodoForm`: 带 modal 包装的完整弹窗

新增侧边栏时复用 TodoFormContent，自动获得表单功能更新。
