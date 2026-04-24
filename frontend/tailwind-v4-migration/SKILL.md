---
name: tailwind-v4-migration
description: Tailwind CSS v4 migration guide for shadcn-ui projects
---

# Tailwind v4 迁移指南

## 核心变更

| 项目 | v3 | v4 |
|------|----|----|
| 包 | `tailwindcss` | `tailwindcss` + `@tailwindcss/vite` |
| 配置 | `tailwind.config.js` + `postcss.config.js` | 仅 `@theme` CSS指令 |
| Vite插件 | 无 | `tailwindcss()` in vite.config.ts |

## 迁移步骤

### 1. 安装依赖

```bash
# 移除旧依赖
pnpm remove tailwindcss postcss autoprefixer

# 安装v4
pnpm add tailwindcss@^4 @tailwindcss/vite@^4
```

### 2. 修改 vite.config.ts

```typescript
import tailwindcss from "@tailwindcss/vite"
import react from "@vitejs/plugin-react"

export default defineConfig({
  plugins: [react(), tailwindcss()],
})
```

### 3. 删除配置文件

```bash
rm tailwind.config.js postcss.config.js
```

### 4. 重写 index.css

```css
@import "tailwindcss";

@theme {
  --color-background: hsl(0 0% 100%);
  --color-foreground: hsl(222.2 84% 4.9%);
  --color-primary: hsl(221.2 83.2% 53.3%);
  --color-primary-foreground: hsl(210 40% 98%);
  /* ... 其他变量 */
}

@layer base {
  * {
    border-color: var(--color-border);
  }
  body {
    background-color: var(--color-background);
    color: var(--color-foreground);
  }
}
```

## 已知兼容性问题

### Zod v4 不兼容 @hookform/resolvers

**问题**: `@hookform/resolvers@5.2.2` 依赖 `zod/v4/core` 导致构建失败

**解决**: 降级到 zod v3

```bash
pnpm add zod@^3.23.8
```

## shadcn init CLI 问题

### 问题
`pnpm dlx shadcn@latest init` 在自动化环境（terminal tool）中会被识别为长驻进程而阻塞

### 解决
不依赖 CLI，直接手动创建/修改配置文件：

1. 从 GitHub 获取模板文件：
   - `https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/package.json`
   - `https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/vite.config.ts`
   - `https://raw.githubusercontent.com/shadcn-ui/ui/main/templates/vite-app/src/index.css`

2. 修改 `components.json` 的 `style` 为 `new-york`

3. 手动创建所需的 UI 组件
