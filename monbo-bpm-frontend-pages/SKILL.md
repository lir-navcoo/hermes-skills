---
name: monbo-bpm-frontend-pages
description: monbo-bpm 前端页面重建 — shadcn/ui + React 19 + TypeScript + Vite 构建修复记录
category: frontend
---

# monbo-bpm 前端页面重建

## 项目信息
- 服务器：101.126.89.23
- 源码路径：`/Users/lirui/monbo-bpm/monbo-bpm-ui/`（本地）
- 部署路径：`/opt/monbo-bpm/ui/`（nginx root）
- 技术栈：React 19 + TypeScript + Vite + shadcn/ui (new-york) + Tailwind v3 + react-hook-form + zod
- npm scripts：`npm run build` → `tsc -b && vite build`

## 关键依赖（package.json 中已安装）
```json
{
  "dependencies": {
    "@hookform/resolvers": "^5.2.2",
    "@radix-ui/react-dialog": "^1.1.15",
    "@radix-ui/react-dropdown-menu": "^2.1.16",
    "@radix-ui/react-label": "^2.1.8",
    "@radix-ui/react-scroll-area": "^1.2.10",
    "@radix-ui/react-select": "^2.2.6",
    "@radix-ui/react-separator": "^2.1.8",
    "@radix-ui/react-slot": "^1.2.4",
    "@radix-ui/react-tabs": "^1.1.13",
    "@radix-ui/react-tooltip": "^1.2.8",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "lucide-react": "^1.8.0",
    "react-hook-form": "^7.72.1",
    "tailwind-merge": "^2.1.0",
    "zod": "^4.3.6",
    "i18next": "latest",
    "react-i18next": "latest",
    "i18next-browser-languagedetector": "latest",
    "zustand": "latest"
  }
}
```

## 必须文件：lib/utils.ts
shadcn 所有组件都依赖 `cn()` 函数，**必须创建**：
```ts
// src/lib/utils.ts
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

## 技术栈（2026-04-22 更新）
- React 19 + TypeScript + Vite + shadcn/ui v4 (new-york style) + Tailwind v4 + react-router-dom v7
- **shadcn v4 关键变化：** 使用 `@base-ui/react` 替代 `@radix-ui/react`，`asChild` prop 在 SidebarMenuButton/SidebarMenuSubButton 上**不可用**
- **国际化：** i18next + react-i18next
- **状态管理：** zustand + persist
- **移动端支持：** shadcn Sidebar 组件 + 内置 Sheet
- **表单验证：** react-hook-form + zod v4

## shadcn v4 路由集成（NavMain + react-router-dom）

**重要：** shadcn v4 的 `SidebarMenuButton` 和 `SidebarMenuSubButton` **不支持 `asChild` prop**，因为底层是 @base-ui/react 而非 @radix-ui/react。

**正确写法：**
```tsx
// 直接菜单项（无需嵌套）
<SidebarMenuButton isActive={location.pathname === item.url} tooltip={item.title}>
  <NavLink to={item.url} className="flex w-full items-center gap-2">
    <item.icon className="size-4" />
    <span>{item.title}</span>
  </NavLink>
</SidebarMenuButton>

// 子菜单项
<SidebarMenuSubButton isActive={location.pathname === subItem.url}>
  <NavLink to={subItem.url} className="flex w-full">
    <span>{subItem.title}</span>
  </NavLink>
</SidebarMenuSubButton>
```

**错误写法（asChild 不存在）：**
```tsx
// ❌ 会报错：Property 'asChild' does not exist
<SidebarMenuButton asChild isActive={...}>
  <NavLink to={...}>...</NavLink>
</SidebarMenuButton>
```

## Tailwind v4 + tsconfig 兼容

```json
{
  "compilerOptions": {
    "ignoreDeprecations": "6.0",
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```
添加 `ignoreDeprecations: "6.0"` 解决 TypeScript 6.0 中 `baseUrl` 被废弃的报错。

## shadcn v4 AppLayout 架构（sidebar-03示例）

### 核心模式
```tsx
import {
  Sidebar,
  SidebarContent,
  SidebarFooter,
  SidebarGroup,
  SidebarHeader,
  SidebarMenu,
  SidebarMenuButton,
  SidebarMenuItem,
  SidebarMenuSub,
  SidebarMenuSubButton,
  SidebarMenuSubItem,
  SidebarProvider,
  SidebarRail,
  SidebarTrigger,
  SidebarInset,
} from '@/components/ui/sidebar';
import { Separator } from '@/components/ui/separator';
import { Outlet, NavLink, useLocation } from 'react-router-dom';

export default function Page() {
  return (
    <SidebarProvider defaultOpen={true}>
      <AppSidebar />
      <SidebarInset>
        <header className="flex h-16 shrink-0 items-center gap-2 border-b px-3">
          <SidebarTrigger />
          <Separator orientation="vertical" className="mr-2 h-4" />
          {/* breadcrumb + right content */}
        </header>
        <main className="flex flex-1 flex-col gap-4 p-4">
          <Outlet />
        </main>
      </SidebarInset>
    </SidebarProvider>
  );
}
```

### 关键点
- `SidebarProvider` 统一管理展开/折叠状态，**不需要手动的useState**
- `SidebarTrigger` 自动处理PC端展开/折叠切换，Mobile端自动弹出Sheet
- `SidebarInset` 包裹Header+main内容区，自动响应sidebar状态
- 移动端：Sheet遮罩自动处理，无需手动控制
- `defaultOpen={true}` 默认展开（PC端）

### AppSidebar 组件结构
```tsx
function AppSidebar({ ...props }: React.ComponentProps<typeof Sidebar>) {
  return (
    <Sidebar {...props}>
      <SidebarHeader>
        <SidebarMenu>
          <SidebarMenuItem>
            <SidebarMenuButton size="lg" asChild>
              <a href="#">
                <div className="flex aspect-square size-8 items-center justify-center rounded-lg bg-sidebar-primary text-sidebar-primary-foreground">
                  <Layers className="size-4" />
                </div>
                <div className="flex flex-col gap-0.5 leading-none">
                  <span className="font-medium">应用名</span>
                  <span className="text-xs text-sidebar-foreground/70">副标题</span>
                </div>
              </a>
            </SidebarMenuButton>
          </SidebarMenuItem>
        </SidebarMenu>
      </SidebarHeader>
      <SidebarContent>
        <SidebarGroup>
          <SidebarMenu>
            {data.navMain.map((item) => (
              <SidebarMenuItem key={item.title}>
                {item.path ? (
                  <SidebarMenuButton asChild isActive={isActive(item.path)}>
                    <NavLink to={item.path}>标题</NavLink>
                  </SidebarMenuButton>
                ) : (
                  <>
                    <SidebarMenuButton asChild>
                      <span className="font-medium">分组标题</span>
                    </SidebarMenuButton>
                    {item.items && (
                      <SidebarMenuSub>
                        {item.items.map((subItem) => (
                          <SidebarMenuSubItem key={subItem.path}>
                            <SidebarMenuSubButton asChild isActive={isActive(subItem.path!)}>
                              <NavLink to={subItem.path!}>子菜单</NavLink>
                            </SidebarMenuSubButton>
                          </SidebarMenuSubItem>
                        ))}
                      </SidebarMenuSub>
                    )}
                  </>
                )}
              </SidebarMenuItem>
            ))}
          </SidebarMenu>
        </SidebarGroup>
      </SidebarContent>
      <SidebarFooter className="border-t p-2" />
      <SidebarRail />
    </Sidebar>
  );
}
```

## shadcn v4 LoginPage 架构（login-03示例）

### Field 组件（必须创建）
```tsx
// src/components/ui/field.tsx
"use client"
import * as React from "react"
import { cn } from "@/lib/utils"
import { Separator } from "@/components/ui/separator"

function FieldGroup({ className, ...props }: React.ComponentProps<"div">) {
  return (
    <div
      data-slot="field-group"
      className={cn("flex w-full flex-col gap-6", className)}
      {...props}
    />
  )
}

function Field({ className, ...props }: React.ComponentProps<"div">) {
  return (
    <div
      data-slot="field"
      className={cn("flex w-full flex-col gap-2", className)}
      {...props}
    />
  )
}

function FieldLabel({ className, htmlFor, ...props }: React.ComponentProps<"label"> & { htmlFor?: string }) {
  return (
    <label
      data-slot="field-label"
      htmlFor={htmlFor}
      className={cn("text-sm font-medium leading-snug", className)}
      {...props}
    />
  )
}

export { Field, FieldGroup, FieldLabel, FieldDescription, FieldSeparator };
```

### Breadcrumb 组件（必须创建）
```tsx
// src/components/ui/breadcrumb.tsx
import * as React from "react"
import { ChevronRight, MoreHorizontal } from "lucide-react"
import { cn } from "@/lib/utils"

function Breadcrumb({ ...props }: React.ComponentProps<"nav">) {
  return <nav aria-label="breadcrumb" data-slot="breadcrumb" {...props} />
}

function BreadcrumbList({ className, ...props }: React.ComponentProps<"ol">) {
  return (
    <ol
      data-slot="breadcrumb-list"
      className={cn("flex flex-wrap items-center gap-1.5 text-sm break-words text-muted-foreground sm:gap-2.5", className)}
      {...props}
    />
  )
}

function BreadcrumbItem({ className, ...props }: React.ComponentProps<"li">) {
  return (
    <li data-slot="breadcrumb-item" className={cn("inline-flex items-center gap-1.5", className)} {...props} />
  )
}

function BreadcrumbLink({ className, ...props }: React.ComponentProps<"a">) {
  return (
    <a data-slot="breadcrumb-link" className={cn("transition-colors hover:text-foreground", className)} {...props} />
  )
}

function BreadcrumbPage({ className, ...props }: React.ComponentProps<"span">) {
  return (
    <span data-slot="breadcrumb-page" role="link" aria-disabled="true" aria-current="page"
      className={cn("font-normal text-foreground", className)} {...props} />
  )
}

function BreadcrumbSeparator({ children, className, ...props }: React.ComponentProps<"li">) {
  return (
    <li data-slot="breadcrumb-separator" className={cn("inline-flex items-center gap-1.5", className)} {...props}>
      {children || <ChevronRight className="size-3.5" />}
    </li>
  )
}

export { Breadcrumb, BreadcrumbList, BreadcrumbItem, BreadcrumbLink, BreadcrumbPage, BreadcrumbSeparator };
```

### LoginPage 结构
```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/card';
import { Field, FieldGroup, FieldLabel } from '@/components/ui/field';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Loader2, Layers } from 'lucide-react';

function LoginFormComponent({ className, ...props }) {
  const { t } = useTranslation();
  const navigate = useNavigate();
  const { login } = useAuthStore();
  const [isLoading, setIsLoading] = useState(false);

  const form = useForm({
    defaultValues: { username: '', password: '' },
  });

  async function onSubmit(data) {
    setIsLoading(true);
    try {
      const result = await authApi.login(data);
      login(result.token, result.username);
      navigate('/');
    } catch (err) {
      // error handling
    } finally {
      setIsLoading(false);
    }
  }

  return (
    <div className={className} {...props}>
      <Card>
        <CardHeader className="text-center">
          <CardTitle className="text-xl">应用名</CardTitle>
          <CardDescription>副标题</CardDescription>
        </CardHeader>
        <CardContent>
          <form onSubmit={form.handleSubmit(onSubmit)}>
            <FieldGroup>
              <Field>
                <FieldLabel htmlFor="username">Username</FieldLabel>
                <Input id="username" type="text" placeholder="Enter username" required disabled={isLoading} {...form.register('username')} />
              </Field>
              <Field>
                <FieldLabel htmlFor="password">Password</FieldLabel>
                <Input id="password" type="password" placeholder="Enter password" required disabled={isLoading} {...form.register('password')} />
              </Field>
              <Field>
                <Button type="submit" className="w-full" disabled={isLoading}>
                  {isLoading ? <Loader2 className="mr-2 h-4 w-4 animate-spin" /> : null}
                  Login
                </Button>
              </Field>
            </FieldGroup>
          </form>
        </CardContent>
      </Card>
    </div>
  );
}

export default function LoginPage() {
  return (
    <div className="flex min-h-svh flex-col items-center justify-center gap-6 bg-muted p-6 md:p-10">
      <div className="flex w-full max-w-sm flex-col gap-6">
        <a href="#" className="flex items-center gap-2 self-center font-medium">
          <div className="flex size-6 items-center justify-center rounded-md bg-primary text-primary-foreground">
            <Layers className="size-4" />
          </div>
          应用名
        </a>
        <LoginFormComponent />
      </div>
    </div>
  );
}
```

## 页面路由
```
/                   → Dashboard（Layout内）
/process-defs       → ProcessDefListPage
/process-insts      → ProcessInstListPage
/tasks              → TaskListPage
/users              → UserListPage
/roles              → RoleListPage
/departments        → DepartmentListPage
```

## 构建失败常见 TS 错误修复

### 1. LoginVO 类型不匹配（TS2339）
后端 `/api/auth/login` 返回 `{token, username}`，前端误以为是 `{token, user: {username}}`。

**修复：** `src/lib/types/index.ts`
```ts
export interface LoginVO {
  token: string;
  username: string;  // 不是 user: User
}
```
同步修复：`LoginPage.tsx` 中 `result.user.username` 改为 `result.username`

### 2. zod schema defaultValues 类型冲突（TS5112）
`status: z.number().default(1)` 与 `defaultValues: { status: 1 }` 冲突。

**修复：** 去掉 `.default()`
```ts
status: z.number(),  // 不是 z.number().default(1)
```

### 3. form.handleSubmit handler 类型（TS2345）
```ts
<form onSubmit={form.handleSubmit(handleSubmit as any)}>
```

### 4. 未使用的 import（TS6133）
React 19 下 `import React from "react"` 不再需要。显式导入但未使用时 TS 报错。

### 5. zod v4 `.extend()` 多余参数（TS2554）
```ts
// 错误
password: z.string().min(1, 'msg', { message: 'msg' })
// 正确
password: z.string().min(1, 'msg')
```

## 部署流程

### 服务器部署（101.126.89.23）
```bash
# 1. 本地构建
cd /Users/lirui/monbo-bpm/monbo-bpm-ui && npm run build

# 2. SCP 到服务器临时目录
expect -c '
spawn scp -o StrictHostKeyChecking=no -r dist root@101.126.89.23:/tmp/monbo-ui-new
expect "password:"
send "Lirui123456\r"
expect eof
'

# 3. SSH 复制到 nginx 目录（逐文件覆盖，不删目录）
expect -c '
spawn ssh root@101.126.89.23 "cp -r /tmp/monbo-ui-new/. /opt/monbo-bpm/ui/"
expect "password:"
send "Lirui123456\r"
expect eof
'

# 4. 验证
curl http://101.126.89.23/
```

**重要：** nginx root 是 `/opt/monbo-bpm/ui/`（不是默认的 `/usr/share/nginx/html/`）

## 服务器信息
- IP: 101.126.89.23
- SSH: root / Lirui123456
- MySQL: root / root123456
- 前端目录: `/opt/monbo-bpm/ui/`
- 后端端口: 8080
- 登录账号: admin / admin123

## 已验证功能（2026-04-22）
- ✅ 登录页（shadcn Card + Form）
- ✅ Dashboard（统计卡片）
- ✅ 用户管理（CRUD + 搜索）
- ✅ 部门管理（CRUD + 树形）
- ✅ 角色管理（CRUD）
- ✅ 流程定义管理
- ✅ 流程实例管理
- ✅ 我的任务
- ✅ 语言切换（中/英）
- ✅ 主题切换（亮/暗）
- ✅ 移动端适配（Sidebar Sheet）

## 后端 API
```
POST   /api/auth/login        {token, username}
GET    /api/auth/info         {username, authorities}
GET    /api/users             分页 {records, total, size, current}
POST   /api/users
PUT    /api/users/:id
DELETE /api/users/:id
GET    /api/roles             分页
POST   /api/roles
PUT    /api/roles/:id
DELETE /api/roles/:id
GET    /api/departments       树形
POST   /api/departments
PUT    /api/departments/:id
DELETE /api/departments/:id
GET    /api/process-defs      分页
POST   /api/process-defs
GET    /api/process-insts      分页
POST   /api/process-insts
GET    /api/tasks             分页
POST   /api/tasks/:id/complete
```
所有请求需要 `Authorization: Bearer <token>` header。
