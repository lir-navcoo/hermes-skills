---
name: shadcn-ui
description: shadcn/ui 组件库完整指南 — 初始化、CLI命令、Theming、Registry、MCP Server、常见坑。写代码时优先使用 shadcn CLI 命令。
category: frontend
tags: [shadcn, react, ui, tailwind, radix-ui]
---

# shadcn/ui 组件库

## 项目感知（自动加载）

**激活条件**：项目根目录有 `components.json` 文件。skill 会自动运行 `shadcn info --json` 获取项目配置并注入上下文。

```bash
# 手动查看当前项目配置
npx shadcn@latest info --json
```

返回内容包括：framework、Tailwind 版本、aliases、base library（radix/base）、icon library、已安装组件列表、文件路径。

**AI 助手使用流程**：
1. 检测到 `components.json` → 自动激活
2. 运行 `shadcn info --json` → 获取项目上下文
3. 使用 `shadcn docs`、`shadcn search` 或 MCP 工具查找组件文档
4. 按照 shadcn 组合规则生成代码（FieldGroup 表单、ToggleGroup 选项集、语义化颜色等）

## 初始化

```bash
# 在项目根目录执行初始化（一键，自动配置 tailwind + components.json）
npx shadcn@latest init

# 指定组件目录（默认 src/components/ui）
npx shadcn@latest init -d

# 完全自定义配置（组件前缀、css 文件路径、aliases）
npx shadcn@latest init --defaults
```

初始化后生成 `components.json` 配置文件。

## CLI 命令（全量参考）

```bash
# 添加组件（核心命令）
npx shadcn@latest add button
npx shadcn@latest add button card input label              # 批量
npx shadcn@latest add button card input label badge table dialog dropdown-menu avatar scroll-area separator tabs tooltip select  # 全部基础组件

# 查看可用的全部组件
npx shadcn@latest add --list

# 搜索组件
npx shadcn@latest search button
npx shadcn@latest search "data table"

# 查看组件文档
npx shadcn@latest docs button
npx shadcn@latest docs dialog

# 查看已安装组件信息
npx shadcn@latest info
npx shadcn@latest info --json

# 升级 shadcn 到最新版本
npx shadcn@latest upgrade

# 移除已添加的组件
npx shadcn@latest remove button

# 查看组件差异
npx shadcn@latest diff button

# 构建（CLI 构建，输出到指定目录）
npx shadcn@latest build --dry-run

# 查看版本
npx shadcn@latest --version

# 初始化时指定样式（default/newspaper）
npx shadcn@latest init --style default
```

## 必须文件：lib/utils.ts

```ts
// src/lib/utils.ts
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

**必须先安装依赖**：
```bash
npm install clsx tailwind-merge
```

## Theming（主题与自定义）

### CSS 变量基础结构

shadcn 使用 CSS 变量系统（`hsl()` 值），定义在入口 CSS 文件（通常是 `src/index.css`）：

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 221.2 83.2% 53.3%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 224.3 76.3% 48%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

### Sidebar 组件 CSS 变量

```css
@layer base {
  :root {
    --sidebar-background: 240 5.3% 96.1%;
    --sidebar-foreground: 240 5.3% 13.8%;
    --sidebar-primary: 240 5.9% 10%;
    --sidebar-primary-foreground: 240 5.3% 98%;
    --sidebar-accent: 240 4.9% 95.9%;
    --sidebar-accent-foreground: 240 5.3% 13.8%;
    --sidebar-border: 240 5.3% 91%;
    --sidebar-ring: 240 5.3% 64.9%;
  }
}
```

### OKLCH 颜色（推荐）

shadcn 官方推荐使用 OKLCH 色彩空间（更广的色域、更自然的渐变）：

```css
--primary: oklch(55% 0.22 251);
```

### Tailwind v3 vs v4

| 特性 | v3 | v4 |
|------|----|----|
| CSS 变量引用 | `hsl(var(--primary))` | `oklch(var(--primary))` |
| 配置方式 | `tailwind.config.js` | CSS `@theme` 块 |
| 暗色模式 | `.dark` class + config | `.dark` class + CSS 变量 |
| Vite集成 | PostCSS (postcss.config.js) | `@tailwindcss/vite` 插件 |
| CLI初始化 | `npx shadcn init` | `npx shadcn init` (same) |

**v4迁移注意**：
- 需要 `tailwindcss` + `@tailwindcss/vite` 两个包
- 不需要 `tailwind.config.js` 和 `postcss.config.js`
- index.css 使用 `@import "tailwindcss"` + `@theme { }` 块
- Vite 插件列表：`plugins: [react(), tailwindcss()]`
- zod v4 与 `@hookform/resolvers` 不兼容，必须用 zod v3

### Tailwind v4 + shadcn Sidebar CSS 变量映射问题

**问题**：shadcn sidebar 组件内部使用 `bg-sidebar` 类，引用 `var(--sidebar-background)`。但 Tailwind v4 的 `@theme` 指令生成的是 `--color-sidebar-background`（带 `color-` 前缀）。这导致 sidebar 背景色无法正确显示。

**原因**：shadcn 组件使用的 CSS 变量命名是 `--sidebar-*`（无 `color-` 前缀），而 Tailwind v4 `@theme` 生成的是 `--color-sidebar-*`。

**解决方案**：在 CSS 中添加变量映射：

```css
@layer base {
  :root {
    /* @theme 生成的变量 */
    --color-sidebar: hsl(0 0% 98%);
    --color-sidebar-foreground: hsl(240 4.8% 95.9%);
    /* ... 其他 sidebar 变量 */

    /* 映射到 shadcn 组件使用的变量名 */
    --sidebar-background: var(--color-sidebar);
    --sidebar-foreground: var(--color-sidebar-foreground);
    --sidebar-primary: var(--color-sidebar-primary);
    --sidebar-primary-foreground: var(--color-sidebar-primary-foreground);
    --sidebar-accent: var(--color-sidebar-accent);
    --sidebar-accent-foreground: var(--color-sidebar-accent-foreground);
    --sidebar-border: var(--color-sidebar-border);
    --sidebar-ring: var(--color-sidebar-ring);
  }

  .dark {
    --color-sidebar: hsl(240 5.9% 10%);
    /* ... 其他 dark 变量 */

    /* Dark mode 映射 */
    --sidebar-background: var(--color-sidebar);
    --sidebar-foreground: var(--color-sidebar-foreground);
    /* ... */
  }
}
```

### Tailwind v4 Sidebar Grid 布局

shadcn sidebar-03 示例使用 CSS Grid 布局实现桌面端两栏结构：

```tsx
<div className="group/sidebar-wrapper ... sidebar-wrapper-grid">
  <AppSidebar />
  <SidebarInset>
    <header>...</header>
    <main>...</main>
  </SidebarInset>
</div>
```

对应的 CSS：

```css
@layer components {
  @media (min-width: 1024px) {
    .sidebar-wrapper-grid {
      display: grid !important;
      grid-template-columns: var(--sidebar-width, 16rem) minmax(0, 1fr) !important;
    }
  }
}
```

**注意**：Tailwind v4 的响应式类（如 `lg:grid`）可能不会为自定义类名生成正确的媒体查询，应使用显式 CSS 规则。

### 暗色模式切换

```bash
# 添加暗色模式
npx shadcn@latest add dark-mode
# 或手动：
npx shadcn@latest add mode-toggle
```

切换机制：`.dark` class 加在 `<html>` 或 `<body>` 上。

## Registry（自定义组件库）

shadcn 支持发布自定义组件库（registry），类似 npm 包但专注于 UI 组件。

### registry.json 结构

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "name": "my-registry",
  "home": "https://my-registry.com",
  "items": [
    {
      "name": "my-component",
      "type": "components:ui",
      "files": ["path/to/MyComponent.tsx"]
    }
  ]
}
```

### 发布流程

1. 构建：`shadcn build --output ./dist`
2. 托管到 GitHub Pages、CDN 或 npm
3. 用户配置 `components.json` 使用你的 registry URL

## MCP Server

shadcn 提供 MCP 服务器，AI 助手可以直接搜索、浏览、安装组件：

```bash
# 安装 MCP 服务器
npx shadcn mcp install
```

MCP 工具包括：`search_components`、`get_component`、`install_component` 等。

## CLI 常见问题

### init 命令在自动化环境挂起

**问题**：`npx shadcn@latest init` 在 Hermes terminal tool 中会被识别为长驻进程而阻塞（即使加了 `--yes` 参数）。

**原因**：shadcn init 启动交互式向导，terminal tool 的进程检测逻辑误判。

**解决**：不依赖 CLI，直接从 GitHub 获取模板文件：

```bash
# 获取 vite-app 模板
curl -sL "https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/package.json"
curl -sL "https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/vite.config.ts"
curl -sL "https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/src/index.css"
curl -sL "https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/src/main.tsx"

# 获取 theme-provider
curl -sL "https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/src/components/theme-provider.tsx"
```

然后手动配置 `components.json`：

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.js",
    "css": "src/index.css",
    "baseColor": "neutral",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

## 常用组件速查

| 组件 | 命令 | 关键 Props |
|------|------|-----------|
| Button | `npx shadcn@latest add button` | `variant`（default/destructive/outline/secondary/ghost/link）, `size`（default/sm/lg/icon） |
| Card | `npx shadcn@latest add card` | Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter |
| Input | `npx shadcn@latest add input` | `type`, `placeholder`, 配合 Label 使用 |
| Label | `npx shadcn@latest add label` | `htmlFor` 关联 Input |
| Badge | `npx shadcn@latest add badge` | `variant`（default/secondary/destructive/outline） |
| Table | `npx shadcn@latest add table` | Table, TableHeader, TableBody, TableRow, TableHead, TableCell, TableCaption |
| Dialog | `npx shadcn@latest add dialog` | Dialog, DialogTrigger, DialogContent, DialogHeader, DialogTitle, DialogDescription, DialogFooter |
| DropdownMenu | `npx shadcn@latest add dropdown-menu` | DropdownMenu, DropdownMenuTrigger, DropdownMenuContent, DropdownMenuItem |
| Avatar | `npx shadcn@latest add avatar` | Avatar, AvatarImage, AvatarFallback |
| ScrollArea | `npx shadcn@latest add scroll-area` | 包裹需要滚动的内容 |
| Separator | `npx shadcn@latest add separator` | `orientation`（horizontal/vertical） |
| Tabs | `npx shadcn@latest add tabs` | Tabs, TabsList, TabsTrigger, TabsContent |
| Select | `npx shadcn@latest add select` | Select, SelectTrigger, SelectValue, SelectContent, SelectItem |
| Tooltip | `npx shadcn@latest add tooltip` | TooltipProvider, Tooltip, TooltipTrigger, TooltipContent |
| Sheet | `npx shadcn@latest add sheet` | 侧边抽屉，替代 Dialog |
| Form | `npx shadcn@latest add form` | 基于 react-hook-form + zod，自动绑定 Input/Label |
| Sidebar | `npx shadcn@latest add sidebar` | SidebarProvider, Sidebar, SidebarHeader, SidebarContent, SidebarFooter, SidebarMenu, SidebarMenuItem, SidebarMenuButton, SidebarInset |
| Data Table | `npx shadcn@latest add data-table` | 基于 TanStack Table，功能完整但复杂 |

## 重要：shadcn v4 使用 Base UI 不是 Radix UI

shadcn/ui v4 底层从 Radix UI 切换到了 **Base UI**（`@base-ui/react`）。这意味着：

- **不能用 Radix 的方式使用组件** — 没有 `asChild` prop
- Base UI 组件使用 `children` slot 模式代替 `asChild`
- 检查方式：`grep "from \"@base-ui" src/components/ui/*.tsx`
- shadcn v4 的 `components.json` 中 `style` 可能是 `base-nova` 或 `new-york`

```tsx
// ❌ 错误（Radix 方式，Base UI 没有 asChild）
<SidebarMenuButton asChild isActive={...}>
  <NavLink to="/">Dashboard</NavLink>
</SidebarMenuButton>

// ✅ 正确（Base UI 方式）
<SidebarMenuButton isActive={...}>
  <NavLink to="/" className="flex w-full">Dashboard</NavLink>
</SidebarMenuButton>
```

### 0. shadcn v4 (Base UI) — button 嵌套 hydration error + MenuGroupRootContext

**背景**：shadcn v4 底层从 Radix 切换到 `@base-ui/react`，Issue #10465（2026-04-22）确认 `asChild` prop 完全不存在。

**问题1**：`DropdownMenuTrigger` 内嵌 `SidebarMenuButton` 会导致 `<button> cannot be a descendant of <button>` hydration error。

**解法**：用 `<div role="button">` 替代 `SidebarMenuButton`，DropdownMenuTrigger 放在 div 外面：

```tsx
// ❌ 错误：button 嵌套 button
<DropdownMenuTrigger>
  <SidebarMenuButton>用户</SidebarMenuButton>
</DropdownMenuTrigger>

// ✅ 正确：DropdownMenuTrigger 独立在外，div 承载内容样式
<DropdownMenu>
  <DropdownMenuTrigger className="w-full">
    <div
      role="button"
      tabIndex={0}
      className="flex items-center gap-2 px-2 py-1.5 text-left w-full cursor-pointer rounded-lg ..."
    >
      <Avatar ... />
      <div className="grid flex-1 text-sm ...">名字</div>
      <ChevronsUpDown className="ml-auto size-4" />
    </div>
  </DropdownMenuTrigger>
  <DropdownMenuContent>...</DropdownMenuContent>
</DropdownMenu>
```

**问题2**：`DropdownMenuLabel` 内部使用 `MenuPrimitive.GroupLabel`，要求必须在 `MenuPrimitive.Group` 内，但 shadcn 旧写法 `Label` 直接放 `Content` 里没有包 `Group`。

**解法**：弃用 `DropdownMenuLabel`，直接用普通 `<div>` 替代。

**切换回 Radix 的代价**：需要降级 shadcn 到 v3.x，重新 init，所有组件重建，breaking changes 巨大，不推荐。

### 1. @fontsource-variable/geist Vite 开发服务器 403

**症状**：控制台报错 `Failed to load resource: the server responded with a status of 403 (Forbidden)` (geist-latin-wght-normal.woff2)

**原因**：Vite 开发服务器拒绝了 pnpm store 中的字体文件路径（路径含 `node_modules/.pnpm-store`）

**修复**：用 Google Fonts CDN 替换

```css
/* ❌ 移除 */
@import "@fontsource-variable/geist";

/* ✅ 替换为 */
@import url("https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap");

/* 同时修改 CSS 变量 */
--font-sans: 'Inter', sans-serif;
```

然后卸载包：`pnpm remove @fontsource-variable/geist`

## 常见坑

### 1. Tailwind v4 sidebar 背景色不显示（CSS 变量映射问题）

**症状**：sidebar 背景色为透明或白色，文字颜色错误。

**原因**：shadcn sidebar 组件使用 `var(--sidebar-background)` 变量，但 Tailwind v4 的 `@theme` 块生成的是 `--color-sidebar-background`。两者命名不匹配。

**排查方法**：在浏览器控制台执行 `getComputedStyle(document.documentElement).getPropertyValue('--sidebar-background')`，如果返回空或 `0 0% 0%`，说明变量未映射。

**修复**：在 `:root` 中添加 `--sidebar-*` 到 `--color-sidebar-*` 的映射（见上文"CSS 变量映射问题"章节）。

### 2. CSS 变量冲突导致样式错乱

**症状**：侧边栏文字不可见、背景色错误。

**原因**：index.css 中重复定义了 sidebar 变量（如 `--sidebar: hsl(240,5.3%,4.9%)` 独立变量），与 shadcn 的 `--sidebar-background` 体系冲突。

**修复**：删除重复的 sidebar 变量块，只保留 `--sidebar-background` 等官方变量体系。

### 2. TypeScript verbatimModuleSyntax 导致 TS1484

```ts
// 错误
import { Handle, Position, NodeProps } from 'reactflow'

// 正确
import { Handle, Position } from 'reactflow'
import type { NodeProps } from 'reactflow'
```

### 3. zod + react-hook-form 类型不匹配

```ts
// 简化 schema，不使用 .default()
const schema = z.object({
  sort: z.number().int().min(0),
  leader: z.string().optional(),
})
const form = useForm({ resolver: zodResolver(schema) as any })
<form onSubmit={form.handleSubmit(handleSubmit as any)}>
```

### 4. 未使用的 import（TS6133）

删除未使用的 import，或在 `tsconfig.json` 中关闭 `noUnusedLocals`（不推荐）。

### 5. React 19 下不需要 `import React`

React 19 中 JSX transform 自动引入，删除 `import React from "react"`。

### 6. Sidebar compound 组件结构

```tsx
<SidebarProvider>
  <Sidebar>
    <SidebarHeader>...</SidebarHeader>
    <SidebarContent>
      <SidebarMenu>
        <SidebarMenuItem>
          <SidebarMenuButton>菜单项</SidebarMenuButton>
        </SidebarMenuItem>
      </SidebarMenu>
    </SidebarContent>
    <SidebarFooter>...</SidebarFooter>
  </Sidebar>
  <SidebarInset>
    {/* 页面内容 */}
  </SidebarInset>
</SidebarProvider>
```

## 构建和部署

```bash
# 1. 开发预览
npm run dev

# 2. 生产构建（必须先通过 tsc -b 类型检查）
npm run build

# 3. 部署构建产物（以 /opt/monbo-bpm/ui 为例）
find /opt/monbo-bpm/ui/ -mindepth 1 -delete   # 清空目标目录
cp -r ./dist/* /opt/monbo-bpm/ui/              # 复制构建产物

# 4. 验证
ls /opt/monbo-bpm/ui/assets/
```

## 官网资源

- 官网：https://ui.shadcn.com
- Skills 文档：https://ui.shadcn.com/docs/skills
- CLI 文档：https://ui.shadcn.com/docs/cli
- Theming 文档：https://ui.shadcn.com/docs/theming
- Registry 文档：https://ui.shadcn.com/docs/registry
- MCP Server：https://ui.shadcn.com/docs/mcp
