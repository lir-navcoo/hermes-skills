---
name: shadcn-ui
description: shadcn/ui 组件库使用指南 — 初始化、添加组件、CSS 变量、常见坑。写代码时优先使用 shadcn CLI 命令。
category: frontend
tags: [shadcn, react, ui, tailwind, radix-ui]
---

# shadcn/ui 组件库

## 初始化

```bash
# 在项目根目录执行初始化（一键，自动配置 tailwind + components.json）
npx shadcn@latest init

# 指定组件目录（默认 src/components/ui）
npx shadcn@latest init -d

# 完全自定义配置（组件前缀、css 文件路径、aliases）
npx shadcn@latest init --defaults
```

初始化后会生成 `components.json` 配置文件。

## 添加组件（核心命令）

```bash
# 添加单个组件，自动写入 src/components/ui/<component>/
npx shadcn@latest add button

# 批量添加多个组件
npx shadcn@latest add button card input label

# 添加所有基础组件
npx shadcn@latest add button card input label badge table dialog dropdown-menu avatar scroll-area separator tabs tooltip select

# 查看可用的全部组件
npx shadcn@latest add --list
```

**注意**：所有组件都依赖 `lib/utils.ts` 中的 `cn()` 函数，初始化后自动生成。

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

## CSS 变量（index.css）

shadcn 要求在入口 CSS 文件（通常是 `src/index.css`）中定义 CSS 变量。以下是完整的基础变量结构：

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
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
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

## Sidebar 组件 CSS 变量

使用 sidebar 组件时需要额外定义 sidebar 变量（浅色主题）：

```css
@layer base {
  :root {
    /* sidebar 组件变量 — 必须有，否则 .bg-sidebar 读取不到正确颜色 */
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

## 常见坑

### 1. CSS 变量冲突导致样式错乱

**症状**：侧边栏文字不可见、背景色错误。

**原因**：index.css 中重复定义了 sidebar 变量（如 `--sidebar: hsl(240,5.3%,4.9%)` 独立变量），与 shadcn 的 `--sidebar-background` 体系冲突。`.bg-sidebar` 类读取的是 `--sidebar-background`，但如果同时定义了 `--sidebar` 这个独立变量且值为深色，就会覆盖正确颜色。

**修复**：删除 index.css 中所有重复的 sidebar 变量块，只保留 shadcn 官方变量体系（`--sidebar-background` 等）。

### 2. TypeScript verbatimModuleSyntax 导致 TS1484

**症状**：`EdgeProps is a type and must be imported using a type-only import`

**原因**：项目 `tsconfig.json` 启用了 `verbatimModuleSyntax`，所有类型必须用 `import type`

**修复**：
```ts
// 错误
import { Handle, Position, NodeProps } from 'reactflow'

// 正确
import { Handle, Position } from 'reactflow'
import type { NodeProps } from 'reactflow'
```

### 3. zod + react-hook-form 类型不匹配

**症状**：`BUILD SUCCESS` 但 `tsc -b` 失败，或 Resolver 类型报错。

**原因**：`z.coerce.number().optional().default()` 与 `zodResolver` 的 `Resolver` 类型不兼容。

**修复**：简化 schema，不使用 `.default()` 链；在 `useForm` 和 `handleSubmit` 处加 `as any` 绕过类型检查：
```ts
const form = useForm({ resolver: zodResolver(schema) as any })
<form onSubmit={form.handleSubmit(handleSubmit as any)}>
```

### 4. 未使用的 import（TS6133）

**症状**：TS 报错 `... is declared but its value is never read`

**修复**：删除未使用的 import，或在 `tsconfig.json` 中关闭 `noUnusedLocals`（不推荐）。

### 5. React 19 下不需要 `import React`

**症状**：`import React from "react"` 触发 TS 警告

**修复**：删除 `import React`，React 19 中 JSX transform 自动引入。

### 6. Sidebar compound 组件结构

使用 sidebar 组件的正确嵌套结构：
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

## 常用组件速查

| 组件 | 命令 | 关键 Props |
|------|------|-----------|
| Button | `npx shadcn@latest add button` | `variant`（default/destructive/outline/secondary/ghose/link）, `size`（default/sm/lg/icon） |
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

## CLI 其他常用命令

```bash
# 升级 shadcn 到最新版本
npx shadcn@latest upgrade

# 移除已添加的组件
npx shadcn@latest remove button

# 查看版本
npx shadcn@latest --version

# 初始化时指定样式（default/newspaper）
npx shadcn@latest init --style default
```
