---
kind: frontend_style
name: llama-ui 前端样式体系：Tailwind v4 + shadcn-svelte + bits-ui
category: frontend_style
scope:
    - '**'
source_files:
    - tools/ui/package.json
    - tools/ui/src/app.css
    - tools/ui/components.json
    - tools/ui/vite.config.ts
    - tools/ui/svelte.config.js
    - tools/ui/src/lib/styles/katex-custom.scss
    - tools/ui/pwa-assets.config.ts
    - tools/ui/pwa-assets-dark.config.ts
    - tools/ui/.storybook/main.ts
    - tools/ui/.storybook/preview.ts
---

## 1. 系统/技术栈

llama.cpp 的前端 UI 位于 `tools/ui/`，是一个独立的 SvelteKit 应用（名称 `llama-ui`），采用以下样式与组件体系：

- **框架**：Svelte 5 + SvelteKit 2（`@sveltejs/kit`）+ Vite 7。
- **CSS 框架**：Tailwind CSS v4（`tailwindcss@4.3.0`），通过 `@import 'tailwindcss'` 在入口引入，并启用 `@source '.'` 扫描整个源码目录以生成原子类。
- **UI 组件库**：shadcn-svelte（通过 `components.json` 的 `https://shadcn-svelte.com/registry` 注册表安装），底层原语来自 `bits-ui`；业务组件位于 `src/lib/components/app/`，基础 UI 组件位于 `src/lib/components/ui/`。
- **变体引擎**：`tailwind-variants`（`tv()` + `VariantProps`）用于声明式地组合 Tailwind 类名和变体（如 `alert.svelte`、`badge.svelte`、`button.svelte` 均使用）。
- **动画**：`tw-animate-css`（通过 `@import 'tw-animate-css'` 引入）提供预置动画。
- **表单/排版插件**：`@tailwindcss/forms`、`@tailwindcss/typography`。
- **主题切换**：基于 `.dark` 类选择器（`@custom-variant dark (&:is(.dark *))`），配合 `mode-watcher` 运行时切换。
- **数学渲染**：KaTeX（通过 `katex-custom.scss` 覆盖字体格式为仅 woff2）。
- **构建产物**：PWA（`@vite-pwa/sveltekit` + `@vite-pwa/assets-generator`），由 `pwa-assets.config.ts` / `pwa-assets-dark.config.ts` 生成图标与启动画面。

## 2. 关键文件

- `tools/ui/package.json` — 依赖与脚本入口（build/dev/test/storybook 等）。
- `tools/ui/src/app.css` — 全局样式入口，Tailwind 导入、设计令牌、暗色主题、滚动条与工具类定义。
- `tools/ui/components.json` — shadcn-svelte 配置（baseColor: `neutral`，路径别名指向 `$lib/components`、`$lib/hooks` 等）。
- `tools/ui/vite.config.ts` / `svelte.config.js` — Vite/SvelteKit 构建配置。
- `tools/ui/src/lib/styles/katex-custom.scss` — KaTeX SCSS 变量覆盖。
- `tools/ui/src/lib/components/ui/*.svelte` — shadcn-svelte 生成的基础组件（alert、button、dialog、drawer、table、select 等）。
- `tools/ui/.storybook/` — Storybook 配置，用于组件文档与视觉回归。

## 3. 架构与约定

### 设计令牌（Design Tokens）
所有颜色、圆角、字体等视觉变量集中在 `src/app.css` 的 `:root` 与 `.dark` 块中，使用 OKLCH 色彩空间定义语义化 CSS 变量（`--background`、`--foreground`、`--primary`、`--destructive`、`--chart-1..5`、`--sidebar-*`、`--code-background` 等），再通过 `@theme inline { --color-*: var(--*) }` 映射到 Tailwind v4 的 theme 命名空间。圆角统一通过 `--radius` 派生 `--radius-sm/md/lg/xl`。

### 主题策略
- 亮色主题在 `:root` 中定义，暗色主题在 `.dark` 中覆盖同一组变量。
- 通过 `@custom-variant dark (&:is(.dark *))` 暴露 `dark:` 前缀。
- 运行时通过 `mode-watcher` 切换 `<html class="dark">` 实现明暗模式。

### 组件分层
- `src/lib/components/ui/`：无业务逻辑的基础原子组件（每个子目录含一个或多个 `.svelte` 与 `index.ts` 导出），由 shadcn-svelte 脚手架生成，遵循 `bits-ui` 原语 + `tailwind-variants` 变体的写法。
- `src/lib/components/app/`：业务组件（聊天、模型管理、设置、对话框等），组合基础组件与状态层（stores/contexts/services）。

### 响应式策略
- 主要使用 Tailwind 断点（如 `@media (min-width: 640px)`）调整 CSS 变量（例如 `--chat-form-area-height`、`--chat-form-area-offset`、`--chat-form-padding-top`），从而让布局随视口变化。
- 移动端与桌面端差异通过媒体查询与 Tailwind 响应式类共同处理。

### 可访问性与交互增强
- `app.css` 在 `@layer base` 中为所有可交互元素（button、checkbox、combobox、menuitem、option、radio、switch、tab、label[for]、select、summary）恢复 `cursor: pointer`，因为 Tailwind v4 移除了默认指针光标。
- 自定义滚动条样式（`scrollbar-width: thin`、hover 时显示半透明 thumb）。
- 支持 `prefers-reduced-motion` 禁用 shimmer 动画。

### 代码风格与格式化
- ESLint（`eslint-plugin-svelte`、`eslint-plugin-perfectionist`、`eslint-plugin-simple-import-sort`）+ Prettier（`prettier-plugin-svelte`、`prettier-plugin-tailwindcss`）。
- 通过 `package.json` 的 `format` / `lint` 脚本统一执行。

## 4. 约定与约束

- **样式入口唯一**：所有 Tailwind 指令与全局样式集中在 `src/app.css`，通过 `@import 'tailwindcss'`、`@plugin '@tailwindcss/forms'`、`@plugin '@tailwindcss/typography'`、`@import 'tw-animate-css'` 引入。
- **设计令牌集中**：所有视觉变量必须定义在 `:root` / `.dark` 的 CSS 变量中，并通过 `@theme inline` 暴露给 Tailwind，禁止在组件内硬编码颜色值。
- **组件样式使用 tailwind-variants**：基础 UI 组件（alert、badge、button 等）通过 `tv()` 声明变体，而非散落的条件类名。
- **shadcn-svelte 组件来源**：新增基础组件应通过 shadcn-svelte CLI 从 `https://shadcn-svelte.com/registry` 拉取，保持与 `components.json` 的路径别名一致。
- **KaTeX 字体限制**：`katex-custom.scss` 强制仅使用 woff2 字体（`$use-woff2: true; $use-woff: false; $use-ttf: false`），以嵌入 bundle 而非外部加载。
- **Storybook 作为组件文档**：组件故事位于 `tests/stories/`，通过 `@storybook/addon-a11y` 进行可访问性检查。
- **PWA 资源生成**：图标与启动画面通过 `pwa-assets-generator` 按 `pwa-assets.config.ts` / `pwa-assets-dark.config.ts` 分别生成，构建脚本 `build-pwa-assets` 是 `build` 的前置步骤。