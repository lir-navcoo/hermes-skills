---
name: monbo-bpm-frontend-pages
description: monbo-bpm 前端页面重建 — shadcn/ui + React 19 + TypeScript + Vite 构建修复记录
category: frontend
---

# monbo-bpm 前端页面重建

## 项目信息
- 服务器：101.126.89.23
- 源码路径：`/Users/lirui/monbo-bpm/monbo-bpm-ui/`（本地）
- 统一仓库：`https://github.com/lir-navcoo/monbo-bpm`（monbo-bpm-api + monbo-bpm-ui 同仓库）
- 部署路径：`/opt/monbo-bpm/ui/`（nginx root）
- 技术栈：React 19 + TypeScript + Vite + shadcn/ui (Base UI) + Tailwind v4 + react-router-dom v7 + zustand
- 推送方式：GitHub REST API 逐文件 PUT（避免 tree API 双重编码问题）
- npm scripts：`pnpm build` → `tsc -b && vite build`

## GitHub 推送（统一仓库 monbo-bpm）
推送 monbo-bpm-ui 到 `monbo-bpm` 仓库的 `main` 分支 `monbo-bpm-ui/` 目录：
```python
import subprocess, os, base64, json

repo = "lir-navcoo/monbo-bpm"
base_path = "/Users/lirui/monbo-bpm/monbo-bpm-ui"
branch = "main"

for root, dirs, files in os.walk(base_path):
    dirs[:] = [d for d in dirs if d not in {'.git', 'node_modules', '.DS_Store', 'dist'}]
    for fname in files:
        fpath = os.path.join(root, fname)
        rel = os.path.relpath(fpath, base_path)
        remote_path = f"monbo-bpm-ui/{rel}"

        with open(fpath, 'rb') as f:
            encoded = base64.b64encode(f.read()).decode('ascii')

        # Check SHA
        check = subprocess.run(
            ['gh', 'api', f'repos/{repo}/contents/{remote_path}?ref={branch}', '--jq', '.sha'],
            capture_output=True, text=True
        )
        sha = check.stdout.strip()

        if check.returncode == 0 and sha:
            subprocess.run(
                ['gh', 'api', '--method', 'PUT', f'repos/{repo}/contents/{remote_path}',
                 '-f', f'message=chore: {remote_path}', '-f', f'content={encoded}', '-f', f'sha={sha}', '-f', f'branch={branch}']
            )
        else:
            subprocess.run(
                ['gh', 'api', '--method', 'PUT', f'repos/{repo}/contents/{remote_path}',
                 '-f', f'message=chore: {remote_path}', '-f', f'content={encoded}', '-f', f'branch={branch}']
            )
```

推送 README.md：
```python
# 先查 SHA，再 PUT
check = subprocess.run(['gh', 'api', f'repos/{repo}/contents/README.md?ref={branch}', '--jq', '.sha'], ...)
sha = check.stdout.strip()
# ... PUT with sha if exists
```

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

**重要：** shadcn v4 的 `SidebarMenuButton` 和 `SidebarMenuSubButton` **不支持 `asChild` prop**，因为底层是 @base-ui/react 而非 @radix-ui/react。`asChild` 根本不存在于该组件。

**正确做法：** 用 `div[role=button]` + `useNavigate` 替代方案（见上方 AppSidebar 菜单项章节）。

**子菜单项：**
```tsx
// SidebarMenuSubButton 同样不能嵌套 a标签，直接用 div 方案
<div
  role="button"
  tabIndex={0}
  onClick={() => handleNavClick(subItem.url)}
  onKeyDown={(e) => e.key === "Enter" && handleNavClick(subItem.url)}
  className="flex w-full ..."
>
  <span>{subItem.title}</span>
</div>
```

**错误写法（asChild 不存在）：**
```tsx
// ❌ 会报错：Property 'asChild' does not exist
<SidebarMenuButton asChild isActive={...}>
  <NavLink to={...}>...</NavLink>
</SidebarMenuButton>
```

## tsconfig 关键坑（TypeScript 6.0.3）

**`baseUrl` 在 TS 6 中被废弃，但 `ignoreDeprecations: "6.0"` 本身在 TS 6.0.3 有 bug（报"值无效"）。**

正确做法：**去掉 `baseUrl`**，Vite 运行时仍然正常解析 `@/` 别名：
```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

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

### AppSidebar 菜单项：禁止嵌套 + useNavigate 路由跳转

**核心原则（已踩坑）：**
1. `SidebarMenuButton` 底层是 `<button>`，不能嵌套 `<a>`（HTML 规范不允许 `button > a`）
2. 不能用 `asChild` prop（Base UI 没有 asChild）
3. 必须用 `div[role=button]` 替代方案，或直接用 `<a>` 标签 + Tailwind 样式

**正确写法（推荐）：用 div[role=button] + useNavigate**
```tsx
import { useLocation, useNavigate } from "react-router-dom";
import { cn } from "@/lib/utils";

function AppSidebar({ ...props }: React.ComponentProps<typeof Sidebar>) {
  const location = useLocation();
  const navigate = useNavigate();

  const handleNavClick = (url: string) => {
    navigate(url, { replace: true }); // replace模式避免浏览器history回退
  };

  return (
    <Sidebar {...props}>
      <SidebarContent>
        {data.navMain.map((group) => (
          <SidebarGroup key={group.title}>
            <SidebarGroupLabel>{group.title}</SidebarGroupLabel>
            <SidebarGroupContent>
              <SidebarMenu>
                {group.items.map((item) => {
                  const active = location.pathname === item.url;
                  return (
                    <SidebarMenuItem key={item.title}>
                      <div
                        role="button"
                        tabIndex={0}
                        onClick={() => handleNavClick(item.url)}
                        onKeyDown={(e) => e.key === "Enter" && handleNavClick(item.url)}
                        data-active={active}
                        className={cn(
                          "peer/menu-button group/menu-button flex w-full cursor-pointer items-center gap-2 overflow-hidden rounded-md p-2 text-left text-sm ring-sidebar-ring outline-hidden hover:bg-sidebar-accent hover:text-sidebar-accent-foreground focus-visible:ring-2 active:bg-sidebar-accent active:text-sidebar-accent-foreground data-active:bg-sidebar-accent data-active:font-medium data-active:text-sidebar-accent-foreground [&>span:last-child]:truncate",
                          active ? "bg-sidebar-accent font-medium text-sidebar-accent-foreground" : ""
                        )}
                      >
                        <span>{item.title}</span>
                      </div>
                    </SidebarMenuItem>
                  );
                })}
              </SidebarMenu>
            </SidebarGroupContent>
          </SidebarGroup>
        ))}
      </SidebarContent>
    </Sidebar>
  );
}
```

**错误写法（已验证不work）：**
```tsx
// ❌ asChild 不存在
<SidebarMenuButton asChild isActive={...}>
  <NavLink to={...}>标题</NavLink>
</SidebarMenuButton>

// ❌ button > a 嵌套非法
<SidebarMenuButton isActive={...}>
  <a href={item.url}>{item.title}</a>
</SidebarMenuButton>

// ❌ w-full justify-start 加在 SidebarMenuButton 上仍然无法点击整行
<SidebarMenuButton className="w-full justify-start" isActive={...}>
  <a href={item.url}>{item.title}</a>
</SidebarMenuButton>
```

**Logo 按钮例外：** Logo 区域仍然用 `SidebarMenuButton size="lg"` 包裹静态内容（不需要跳转），不影响。

### Sidebar + 固定Header + 内容滚动布局

**需求：** Header 固定不滚动，内容区超出时内部滚动。

**完整布局链路（layout.tsx）：**
```tsx
<SidebarProvider defaultOpen={true}>
  <AppSidebar />
  <SidebarInset className="overflow-hidden h-full">  {/* 关键：h-full + overflow-hidden */}
    <LayoutHeader className="flex-shrink-0" />   {/* 固定顶部 */}
    <main className="flex-1 overflow-y-auto">   {/* 内容区滚动 */}
      <Outlet />
    </main>
  </SidebarInset>
</SidebarProvider>
```

**关键点：**
- `SidebarProvider` 的 `div` 有 `min-h-svh`，但 `SidebarInset`（`<main>`）默认没有高度约束
- `SidebarInset` 加 `h-full` 让它填满父级高度，建立明确的 flex 链路
- `overflow-hidden` 防止 `SidebarInset` 自身滚动
- `LayoutHeader` 加 `flex-shrink-0` 禁止收缩
- `main` 用 `flex-1 overflow-y-auto` 占满剩余空间并内部滚动

**LayoutHeader 组件需要接受 className prop：**
```tsx
// layout-header.tsx
export function LayoutHeader({ className }: { className?: string }) {
    return (
        <header className={"flex h-16 shrink-0 items-center gap-2 border-b " + className}>
            ...
        </header>
    )
}
```

**核心原则（已踩坑）：**
1. `SidebarMenuButton` 底层是 `<button>`，不能嵌套 `<a>`（HTML 规范不允许 `button > a`）
2. 不能用 `asChild` prop（Base UI 没有 asChild）
3. 必须用 `div[role=button]` 替代方案，或直接用 `<a>` 标签 + Tailwind 样式

**正确写法（推荐）：用 div[role=button] + useNavigate**
```tsx
import { useLocation, useNavigate } from "react-router-dom";
import { cn } from "@/lib/utils";

function AppSidebar({ ...props }: React.ComponentProps<typeof Sidebar>) {
  const location = useLocation();
  const navigate = useNavigate();

  const handleNavClick = (url: string) => {
    navigate(url, { replace: true }); // replace模式避免浏览器history回退
  };

  return (
    <Sidebar {...props}>
      <SidebarContent>
        {data.navMain.map((group) => (
          <SidebarGroup key={group.title}>
            <SidebarGroupLabel>{group.title}</SidebarGroupLabel>
            <SidebarGroupContent>
              <SidebarMenu>
                {group.items.map((item) => {
                  const active = location.pathname === item.url;
                  return (
                    <SidebarMenuItem key={item.title}>
                      <div
                        role="button"
                        tabIndex={0}
                        onClick={() => handleNavClick(item.url)}
                        onKeyDown={(e) => e.key === "Enter" && handleNavClick(item.url)}
                        data-active={active}
                        className={cn(
                          "peer/menu-button group/menu-button flex w-full cursor-pointer items-center gap-2 overflow-hidden rounded-md p-2 text-left text-sm ring-sidebar-ring outline-hidden hover:bg-sidebar-accent hover:text-sidebar-accent-foreground focus-visible:ring-2 active:bg-sidebar-accent active:text-sidebar-accent-foreground data-active:bg-sidebar-accent data-active:font-medium data-active:text-sidebar-accent-foreground [&>span:last-child]:truncate",
                          active ? "bg-sidebar-accent font-medium text-sidebar-accent-foreground" : ""
                        )}
                      >
                        <span>{item.title}</span>
                      </div>
                    </SidebarMenuItem>
                  );
                })}
              </SidebarMenu>
            </SidebarGroupContent>
          </SidebarGroup>
        ))}
      </SidebarContent>
    </Sidebar>
  );
}
```

**错误写法（已验证不work）：**
```tsx
// ❌ asChild 不存在
<SidebarMenuButton asChild isActive={...}>
  <NavLink to={...}>标题</NavLink>
</SidebarMenuButton>

// ❌ button > a 嵌套非法
<SidebarMenuButton isActive={...}>
  <a href={item.url}>{item.title}</a>
</SidebarMenuButton>

// ❌ w-full justify-start 加在 SidebarMenuButton 上仍然无法点击整行
<SidebarMenuButton className="w-full justify-start" isActive={...}>
  <a href={item.url}>{item.title}</a>
</SidebarMenuButton>
```

**Logo 按钮例外：** Logo 区域仍然用 `SidebarMenuButton size="lg"` 包裹静态内容（不需要跳转），不影响。

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

### 3. 后端响应 `{code, message, data}` 格式 + 错误处理
后端所有接口统一返回 `{code, message, data}`，`extractData()` 不仅提取 data，还要判断 code 非 200 时抛出错误：
```typescript
// src/lib/api/index.ts
function extractData(res: any): any {
  const code = res?.data?.code;
  const message = res?.data?.message;
  if (code !== 200) {
    const err = new Error(message || "请求失败");
    (err as any).code = code;
    throw err;
  }
  return res?.data?.data ?? res?.data ?? null;
}

// 所有 API 函数都通过 extractData 处理响应，错误会被统一抛出
// 调用处 catch(err: any) 可获取 err.message 作为后端错误提示
export async function updateDepartment(id: number, data: DepartmentFormData): Promise<void> {
  const res = await api.put(`/api/departments/${id}`, data);
  extractData(res); // code !== 200 时抛错，不再静默忽略
}
```

### 4. 类型渲染保护
`stats?.[key]` 可能返回对象，渲染前必须检查类型：
```tsx
{typeof stats?.[key] === "number" ? stats?.[key] : "-"}
```

### 5. verbatimModuleSyntax
TS 严格模式下，类型导入必须分离：
```tsx
import { fetchXxx } from "@/lib/api";
import type { XxxType } from "@/lib/api";
```

### 6. 电话号码格式验证（支持手机 + 固定电话）
```typescript
// 正则：支持 1开头手机号 / 010-xxxxxxxx / 021xxxxxxx 等固定电话
/^(1[3-9]\d{9}|0\d{2,3}-?\d{7,8})$/

// input 下方实时校验提示
{form.phone && !/^(1[3-9]\d{9}|0\d{2,3}-?\d{7,8})$/.test(form.phone) && (
  <p className="text-destructive text-xs">格式有误，请输入有效手机号或固定电话</p>
)}

// 保存按钮也要校验
if (form.phone && !/^(1[3-9]\d{9}|0\d{2,3}-?\d{7,8})$/.test(form.phone)) return
```
TS 严格模式下，类型导入必须分离：
```tsx
import { fetchXxx } from "@/lib/api";
import type { XxxType } from "@/lib/api";
```

### 6. TypeScript 6.0 baseUrl 废弃
`baseUrl` 在 TS 6 被标记为废弃，但 `ignoreDeprecations: "6.0"` 在 TS 6.0.3 有 bug（报错"值无效"）。解法：直接去掉 `baseUrl`，Vite 运行时仍然正常解析 `@/` 别名。

### 7. Sonner 依赖
`next-themes` 是 shadcn sonner 的隐式 peer dependency，需已安装。

### 8. Base UI 通用 asChild → render 替换规则（重要！）

**shadcn v4 使用 `@base-ui/react`，所有 Radix UI 的 `asChild` prop 在 Base UI 中改为 `render`。**

| 组件 | Radix 写法 | Base UI 写法 |
|------|-----------|-------------|
| `SidebarMenuButton` | `asChild` | `div[role=button]` + `useNavigate`（见上方章节） |
| `SidebarMenuSubButton` | `asChild` | `div[role=button]` + `useNavigate` |
| `DropdownMenuTrigger` | `asChild` | `render={<Button />}` 或直接用 `className` |
| `TooltipTrigger` | `asChild` | `render={<span />}` 或直接用 |
| `SelectTrigger` | `asChild` | 直接渲染，不需要 `asChild` |

**DropdownMenuTrigger 正确写法（直接用 className，不需要 asChild）：**
```tsx
<DropdownMenuTrigger className="flex items-center gap-1 rounded border border-input bg-background px-2 hover:bg-accent hover:text-accent-foreground cursor-pointer text-sm">
  <IconLayoutColumns className="size-4" />
  列设置
</DropdownMenuTrigger>
```

**TooltipTrigger 正确写法（不需要 asChild/render）：**
```tsx
// shadcn Tooltip 组件的 TooltipTrigger 通常直接包裹内容，不需要 asChild
<Tooltip>
  <TooltipTrigger asChild>
    <Button>...</Button>  // ❌ asChild 不存在
  </TooltipTrigger>
</Tooltip>

// 正确写法
<Tooltip>
  <TooltipTrigger>
    <Button>...</Button>  // ✅ 直接渲染
  </TooltipTrigger>
</Tooltip>
```

**Select 组件 onValueChange TypeScript 修复：**
```tsx
// ❌ TypeScript 报错：Argument of type 'string' is not assignable to parameter of type '""'
onValueChange={setSelectedDeptId}

// ✅ 修复：确保空值时返回空字符串
onValueChange={v => setSelectedDeptId(v || 'all')}
onValueChange={v => setSelectedProcessDefId(v || '')}

// Select 的 value 类型是 string（不是 string | undefined），空字符串需要显式处理
```

### 9. .old.tsx 文件处理

重构过程中产生的 `.old.tsx` 文件（如 `UserListPage.old.tsx`）在确认无引用后应删除：
```bash
# 确认无引用
grep -r "UserListPage.old" src/ --include="*.tsx" --include="*.ts"

# 删除
rm src/pages/users/UserListPage.old.tsx
```

### 9. 树形结构表格中 filterTree 的 TypeScript 类型问题

**问题场景：** 部门树形表格中，用 `.map().filter()` + 类型谓词过滤空值时，TS2322 报错 `Type 'null' is not assignable to type 'Department'`。

**错误写法：**
```tsx
// ❌ TypeScript 无法正确收窄 Department | null 类型
const filterTree = (list: Department[], keyword: string): Department[] => {
  return list
    .map((dept) => {
      // ...
      return matched ? { ...dept, children: filteredChildren } : null
    })
    .filter((d): d is Department => d !== null) // TS2322 报错
}
```

**正确写法（用 for 循环）：**
```tsx
// ✅ TypeScript 能正确推断类型
const filterTree = (list: Department[], keyword: string): Department[] => {
  const result: Department[] = []
  for (const dept of list) {
    const matched = ...
    const filteredChildren = filterTree(dept.children || [], keyword)
    if (matched || filteredChildren.length > 0) {
      result.push({ ...dept, children: filteredChildren })
    }
  }
  return result
}
```

## 流程设计器（ProcessDesigner）forwardRef 多操作暴露模式

### 场景
设计器需要外部调用多个操作（保存/导入/导出/撤销/重做/缩放等），且需要隐藏内置工具栏。

### Props 接口
```typescript
interface ProcessDesignerProps {
  initialData?: ProcessDefinition
  onSave?: (data: ProcessDefinition) => void
  imperativeRef?: RefObject<{
    triggerSave: () => void
    triggerExport: () => void
    triggerImport: () => void
    triggerUndo: () => void
    triggerRedo: () => void
    triggerZoomIn: () => void
    triggerZoomOut: () => void
    triggerFitView: () => void
  }>
  hideToolbar?: boolean
}
```

### useImperativeHandle 位置规则（重要！）
**`useImperativeHandle` 必须放在所有 handler 定义之后**，否则引用 undefined。

正确顺序：
```typescript
// 1. 所有 handler 定义
const handleSave = useCallback(...)
const handleExport = useCallback(...)
const handleImport = useCallback(...)
const handleUndo = useCallback(...)
const handleRedo = useCallback(...)
const handleZoomIn = useCallback(...)
const handleZoomOut = useCallback(...)
const handleFitView = useCallback(...)

// 2. useImperativeHandle 放最后
useImperativeHandle(imperativeRef, () => ({
  triggerSave: handleSave,
  triggerExport: handleExport,
  triggerImport: handleImport,
  triggerUndo: handleUndo,
  triggerRedo: handleRedo,
  triggerZoomIn: handleZoomIn,
  triggerZoomOut: handleZoomOut,
  triggerFitView: handleFitView,
}), [handleSave, handleExport, handleImport, handleUndo, handleRedo, handleZoomIn, handleZoomOut, handleFitView])
```

### forwardRef 类型声明
```typescript
const ProcessDesigner = forwardRef<{
  triggerSave: () => void
  triggerExport: () => void
  triggerImport: () => void
  triggerUndo: () => void
  triggerRedo: () => void
  triggerZoomIn: () => void
  triggerZoomOut: () => void
  triggerFitView: () => void
}, ProcessDesignerProps>((props, ref) => {
  return (
    <ReactFlowProvider>
      <ProcessDesignerInner
        {...props}
        imperativeRef={ref as unknown as RefObject<{ /* 同上 */ }>}
      />
    </ReactFlowProvider>
  )
})
```

### 设计器页面使用示例（顶栏合并所有工具）
```tsx
// 页面 ref
const designerRef = useRef<{
  triggerSave: () => void
  triggerExport: () => void
  triggerImport: () => void
  triggerUndo: () => void
  triggerRedo: () => void
  triggerZoomIn: () => void
  triggerZoomOut: () => void
  triggerFitView: () => void
}>(null)

// 顶栏按钮直接调用
<Button onClick={() => designerRef.current?.triggerSave()}>保存</Button>
<Button onClick={() => designerRef.current?.triggerExport()}>导出</Button>
<Button onClick={() => designerRef.current?.triggerUndo()}>撤销</Button>

// 设计器组件
<ProcessDesigner
  ref={designerRef}
  hideToolbar  // 隐藏内置工具栏
  onSave={async (data) => { /* ... */ }}
/>
```

---

## 部署流程

### 服务器部署（101.126.89.23）

**方式一：rsync（推荐，增量同步）**
```bash
# 本地构建
cd /Users/lirui/monbo-bpm/monbo-bpm-ui && pnpm build

# rsync 同步（expect 自动输入密码）
expect -c '
set timeout 300
spawn rsync -az --delete /Users/lirui/monbo-bpm/monbo-bpm-ui/dist/ root@101.126.89.23:/opt/monbo-bpm/ui/dist/
expect "password:" { send "Lirui123456\r" }
expect eof
'
```

**方式二：scp（首次或全量覆盖）**
```bash
# 1. 本地构建
cd /Users/lirui/monbo-bpm/monbo-bpm-ui && pnpm build

# 2. SCP 到服务器
expect -c '
spawn scp -o StrictHostKeyChecking=no -r /Users/lirui/monbo-bpm/monbo-bpm-ui/dist root@101.126.89.23:/tmp/monbo-ui-new/
expect "password:"
send "Lirui123456\r"
expect eof
'

# 3. SSH 复制到 nginx 目录
expect -c "
spawn ssh root@101.126.89.23 \"cp -r /tmp/monbo-ui-new/. /opt/monbo-bpm/ui/\"
expect \"password:\"
send \"Lirui123456\r\"
expect eof
"

# 4. 验证
curl http://101.126.89.23/
```

**注意：**
- nginx root 是 `/opt/monbo-bpm/ui/`（不是默认的 `/usr/share/nginx/html/`）
- 服务器 SSH 只接受公钥认证，密码 `Lirui123456` 仅用于 expect 自动化脚本

## 服务器信息
- IP: 101.126.89.23
- SSH: root / Lirui123456
- MySQL: root / root123456
- 前端目录: `/opt/monbo-bpm/ui/`
- 后端端口: 8080
- 登录账号: admin / admin123

## 部门树形表格关键模式（departments/data-table.tsx）

### flattenTree + expandedIds 展开逻辑

```typescript
// 扁平化树结构，同时记录层级；只输出当前展开的节点
function flattenTree(tree: Department[], expandedIds: Set<number>, level = 0): FlatNode[] {
  const result: FlatNode[] = []
  for (const dept of tree) {
    const children = dept.children ?? []
    result.push({ dept, level, hasChildren: children.length > 0 })
    // 只有当前节点在 expandedIds 中，才继续扁平化子节点
    if (children.length > 0 && expandedIds.has(dept.id)) {
      result.push(...flattenTree(children, expandedIds, level + 1))
    }
  }
  return result
}
```

### 初始展开逻辑（重要！）

**初始 `expandedIds` 应该存根节点 ID，而不是一级子节点 ID。**

因为 `flattenTree` 的规则是"父节点在 `expandedIds` 中才展示子节点"：
- 根节点 ID 在 `expandedIds` → 一级子部门被展示
- 一级子节点 ID 在 `expandedIds` → 二级子部门被展示

**关键坑：`useState` 的 lazy initializer 只在首次挂载时执行一次。**

如果这样写：
```typescript
// ❌ 错误：data=[] 时 initialExpandedIds = new Set([])，API返回后 data 变了但 state 不更新
const initialExpandedIds = React.useMemo(() => {
  return new Set<number>(data.map((d) => d.id))
}, [data])
const [expandedIds, setExpandedIds] = React.useState<Set<number>>(initialExpandedIds)
```

正确写法：用 `useEffect` 同步：
```typescript
// ✅ 正确：data 变化时通过 useEffect 同步 expandedIds
const [expandedIds, setExpandedIds] = React.useState<Set<number>>(() => new Set())

React.useEffect(() => {
  const ids = new Set<number>()
  for (const root of data) {
    ids.add(root.id)
  }
  setExpandedIds(ids)
}, [data])
```

### 父部门选择器（flatDeptOptions + SelectValue 受控模式）

```typescript
// 扁平化部门树，用于父部门选择器选项（排除自身，避免循环引用）
function flatDeptOptions(tree: Department[], level = 0, excludeId?: number): { id: number; label: string }[] {
  const result: { id: number, label: string }[] = []
  for (const dept of tree) {
    if (excludeId !== undefined && dept.id === excludeId) continue
    result.push({ id: dept.id, label: "　".repeat(level) + dept.deptName })
    if (dept.children?.length) {
      result.push(...flatDeptOptions(dept.children, level + 1, excludeId))
    }
  }
  return result
}

// SelectValue 显示选中项的文本（不是 placeholder）
<SelectValue>
  {form.parentId === null || form.parentId === undefined
    ? "无（顶级部门）"
    : (() => {
        const findLabel = (list: Department[], id: number): string => {
          for (const d of list) {
            if (d.id === id) return d.deptName
            if (d.children?.length) {
              const found = findLabel(d.children, id)
              if (found) return found
            }
          }
          return ""
        }
        return findLabel(tree, form.parentId) || "无（顶级部门）"
      })()}
</SelectValue>
```

### 状态 Badge + 图标（避免列设置取消显示关键列）

```typescript
// 部门名称列：enableHiding: false（不可从列设置中隐藏）
{ accessorKey: "deptName", enableHiding: false, ... }

// 状态列：用 Badge + 图标
{
  accessorKey: "status",
  header: "状态",
  cell: ({ row }) => row.original.dept.status === 1
    ? <Badge className="bg-green-100 text-green-700"><IconCircleCheckFilled className="size-3 mr-1"/>启用</Badge>
    : <Badge className="bg-red-100 text-red-700"><IconX className="size-3 mr-1"/>停用</Badge>,
}
```

## 已验证功能（2026-04-22）
- ✅ 登录页（shadcn Card + Form）
- ✅ Dashboard（统计卡片）
- ✅ 用户管理（CRUD + 搜索）
- ✅ 部门管理（CRUD + 树形）
- ✅ 角色管理（CRUD）
- ✅ 流程定义管理
- ✅ 流程实例管理
- ✅ 我的任务
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
