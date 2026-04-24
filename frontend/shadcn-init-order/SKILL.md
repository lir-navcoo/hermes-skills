---
name: shadcn-init-order
description: shadcn init requires Tailwind CSS installed first
---

# shadcn init 安装顺序

## 问题
shadcn init 报错：
```
✖ Validating Tailwind CSS.
No Tailwind CSS configuration found at /path/to/project.
```

## 原因
shadcn init 要求 Tailwind CSS 必须先安装并配置好。

## 正确顺序

### 1. 安装 Tailwind CSS（先装）
```bash
pnpm add tailwindcss @tailwindcss/vite
```

### 2. 配置 vite.config.ts（后续步骤）
```ts
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export default defineConfig({
  plugins: [tailwindcss()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

### 3. 创建基础 CSS 文件（后续步骤）
```css
@import "tailwindcss";
```

### 4. 配置 tsconfig（后续步骤）
确保 `tsconfig.app.json` 中配置了 `@` alias

### 5. 然后再运行 shadcn init
```bash
npx shadcn@latest init --defaults
# 或交互式
npx shadcn@latest init
```

## 关键点
- shadcn init 必须在 Tailwind CSS 安装之后执行
- 如果顺序错了，先装 Tailwind，再重新运行 shadcn init

## 重要发现：pnpm dlx vs npx

`pnpm dlx shadcn@latest init` 会失败，报错：
```
ERR_PNPM_NO_IMPORTER_MANIFEST_FOUND No package.json was found in the shadcn cache directory
```

**正确做法**：使用 `npx shadcn@latest init --defaults`（非交互式，自动使用默认值）

```bash
npx shadcn@latest init --defaults
```

交互式方式（需要手动选择）：
```bash
npx shadcn@latest init
# 选择 Radix → Nova → 默认选项...
```
