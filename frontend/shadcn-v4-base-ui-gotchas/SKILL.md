---
name: shadcn-v4-base-ui-gotchas
description: shadcn v4 (Base UI) 踩坑记录 - asChild不存在、button嵌套、DropdownMenuLabel context问题
tags: [shadcn, base-ui, frontend]
created: 2026-04-22
---

# shadcn v4 Base UI 踩坑记录

## 背景
shadcn v4 (style=base-nova) 已切换至 Base UI，Radix 已完全移除。components.json 中 style 为 `base-nova`。

---

## 1. asChild 不存在

**问题：** Radix 时代的 `asChild` prop 在 Base UI 中不存在。

**报错：**
```
React does not recognize the `asChild` prop on a DOM element.
```

**解法：** Base UI 没有插槽模式，组合方式改用 Hooks API：
- `useDropdownMenu` → 返回 `rootProps`, `triggerProps`, `menuProps`
- 触发区用 `<div {...triggerProps}>` 代替 Radix 的 `asChild` 模式

---

## 2. `<button>` 嵌套问题（NavUser 场景）

**问题：** `DropdownMenuTrigger` 渲染为 `<button>`，`SidebarMenuButton` 内部也是 `<button>`，形成非法嵌套。

**报错：** `In HTML, <button> cannot be a descendant of <button>.`

**最简解法：** 用 `<div role="button" tabIndex={0}>` 替代 `SidebarMenuButton`：
```tsx
<DropdownMenuTrigger className="w-full">
  <div
    role="button"
    tabIndex={0}
    className="flex items-center gap-2 px-2 py-1.5 text-left w-full cursor-pointer rounded-lg ..."
  >
    <Avatar ... />
    <div className="grid flex-1 text-sm ...">...</div>
    <ChevronsUpDown className="ml-auto size-4" />
  </div>
</DropdownMenuTrigger>
```

---

## 3. DropdownMenuLabel 必须包在 DropdownMenuGroup 内

**问题：** `DropdownMenuLabel` 底层使用 `MenuPrimitive.GroupLabel`，Base UI 要求 GroupLabel 必须在 Group context 内。

**报错：** `Base UI: MenuGroupRootContext is missing.`

**解法：** 弃用 `DropdownMenuLabel`，直接用 `<div>` 替代：
```tsx
// 错误 ❌
<DropdownMenuContent>
  <DropdownMenuLabel>标题</DropdownMenuLabel>
</DropdownMenuContent>

// 正确 ✅
<DropdownMenuContent>
  <div className="px-1 py-1.5 text-sm font-medium">标题</div>
</DropdownMenuContent>
```

---

## 4. TypeScript 6 + baseUrl deprecation

**问题：** TypeScript 6 中 `baseUrl` 被废弃，`tsc -b` 报错。TS 6.0.3 中 `ignoreDeprecations: "6.0"` 字符串格式也有 bug，同样报错。

**解法：** 直接去掉 `baseUrl`，Vite 在运行时已配置 `@` 别名解析，不需要 `baseUrl`：
```json
// tsconfig.app.json - 删除 baseUrl
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

---

## 5. 后端响应格式 `{code, message, data}`

**问题：** 后端统一返回 `{code, message, data}` 包装格式，直接当数据用会报错 "Objects are not valid as a React child"。

**解法：** 封装 `extractData` helper：
```typescript
// api.ts
function extractData(res: any): any {
  return res?.data?.data ?? res?.data ?? null;
}

// 使用
const res = await api.get("/api/users/count").catch(() => ({ data: { data: 0 } }));
const count = extractData(res); // 0
```

## 6. API返回非数组时 map 报错

**问题：** `recentProcs.map is not a function` —— API 返回 `{records: [...]}` 或直接返回 `null` 而非数组。

**解法：** 加 `Array.isArray` 保护：
```typescript
const data = extractData(res);
return Array.isArray(data) ? data : [];
```

---

## 8. SidebarProvider 高度链路与主内容区滚动

**问题：** `LayoutHeader` 固定 + `<main>` 内容区滚动失效，整体一起滚。

**布局链路（原始状态）：**
```
SidebarProvider (div, min-h-svh)  ← 根容器
  └── AppSidebar (fixed, 不占空间)
  └── SidebarInset (main, flex-1 flex-col)
        ├── LayoutHeader (header, h-16)
        └── main (flex-1 overflow-y-auto)  ← 不滚动
```

**根因 1：`min-h-svh` 是最小高度，不是固定高度。**
`SidebarProvider` 的 wrapper 是 `flex min-h-svh w-full`。在 flexbox 中，`min-height` **不约束** flex 子元素——子元素可以超出容器高度。此时 `SidebarInset` 的 `flex-1` 没有确定的上界，`overflow-y-auto` 无效。

**根因 2：`SidebarInset` 缺少高度锚点。**
`SidebarInset` 的 class 默认只有 `flex-1 flex-col`，没有 `h-full`，它的 flex 行为不受控。

**修复（完整高度链路）：**
```tsx
// index.css
body {
  @apply overflow-hidden h-screen;  // 禁止body滚动，固定视口高度
}

// layout.tsx
<SidebarProvider defaultOpen={true} className="h-screen overflow-hidden">  {/* 固定高度，阻断min-height传递 */}
  <AppSidebar />
  <SidebarInset className="flex flex-col h-full overflow-hidden">   {/* 填满Provider */}
    <LayoutHeader className="flex-shrink-0"/>                           {/* 固定顶部 */}
    <main className="flex-1 p-4 overflow-y-auto">                       {/* 占满剩余空间，内部滚动 */}
      <Outlet />
    </main>
  </SidebarInset>
</SidebarProvider>
```

**关键要点：**
- `body`: `h-screen overflow-hidden` — 固定视口，禁止页面整体滚动
- `SidebarProvider`: `h-screen overflow-hidden` — 用固定高度替换 `min-h-svh`，阻断其向上传递
- `SidebarInset`: `h-full` — 填满 Provider 的固定高度
- `LayoutHeader`: `flex-shrink-0` — 禁止被压缩
- `main`: `flex-1 overflow-y-auto` — 占满剩余空间后内部滚动

**为什么 `h-full` 加在 `SidebarProvider` 而非 `SidebarInset`？**
因为 `SidebarProvider` 的 wrapper DOM 元素是 `div`，其自身没有高度约束——它的子元素（SidebarInset）即使设置 `h-full` 也只会相对于 `div` 生效，而 `div` 由 `min-h-svh` 决定。必须在 `SidebarProvider` 这一层建立固定高度锚点。

## 9. 后端 code 非 200 时 extractData 未抛出错误

**问题：** `updateDepartment` 调用后返回 `code=400`，但页面显示"更新成功"。`extractData` 只取 data 不判断 code。

**解法：**
```typescript
// api.ts
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

// 调用的地方也要改 catch
} catch (err: any) {
  toast.error(err.message || "更新失败");
}
```

---

## 关键文件

**问题：** `SidebarMenuButton` 底层是 `<button>`，内部包 `<a>` 标签形成非法嵌套，且 `w-full justify-start` class 无法让按钮占满整行。

**现象：** 只有点击文字才能跳转，点击文字外的按钮区域无反应。

**Solution A（无效）：** 给 `SidebarMenuButton` 加 `className="w-full justify-start"` —— 无效。

**Solution B 升级版（推荐）：** 用 `<div role="button">` + `useNavigate` 路由跳转，替代 `<a>` 标签：
```tsx
// 错误 ❌
<SidebarMenuButton isActive={active}>
  <a href={item.url}>{item.title}</a>
</SidebarMenuButton>

// 正确 ✅ - 完整实现
import { useLocation, useNavigate } from "react-router-dom"
import { cn } from "@/lib/utils"

function isRouteActive(pathname: string, itemUrl: string): boolean {
  if (itemUrl === "/") return pathname === "/"
  return pathname === itemUrl
}

// 在组件内
const location = useLocation()
const navigate = useNavigate()
const active = isRouteActive(location.pathname, item.url)

<div
  role="button"
  tabIndex={0}
  onClick={() => navigate(item.url, { replace: true })}
  onKeyDown={(e) => e.key === "Enter" && navigate(item.url, { replace: true })}
  data-active={active}
  className={cn(
    "peer/menu-button group/menu-button flex w-full cursor-pointer items-center gap-2 " +
    "overflow-hidden rounded-md p-2 text-left text-sm ring-sidebar-ring " +
    "outline-hidden transition-[width,height,padding] " +
    "hover:bg-sidebar-accent hover:text-sidebar-accent-foreground " +
    "focus-visible:ring-2 active:bg-sidebar-accent " +
    "data-active:bg-sidebar-accent data-active:font-medium " +
    "data-active:text-sidebar-accent-foreground " +
    "[&>span:last-child]:truncate",
    active ? "bg-sidebar-accent font-medium text-sidebar-accent-foreground" : ""
  )}
>
  <span>{item.title}</span>
</div>
```
**要点：**
- `replace: true` 防止浏览器后退键回退到上一个历史记录
- `tabIndex={0}` + `onKeyDown` 保证键盘可访问
- `cursor-pointer` 提供视觉反馈

---

## 10. Select 组件显示文本问题（shadcn v4 高频坑）

**问题：** `SelectValue` 使用 `placeholder` 属性时，已选项的文本显示不正确；编辑弹窗中部门/角色/状态选择后显示空白或错误文本。

**根因：** shadcn v4（Base UI）的 `SelectValue` 的 `placeholder` 属性和自动派生选中项文本的行为不可靠，不能依赖它正确显示已选项。

**解法：** 不用 `placeholder`，改为在 `SelectValue` 内直接内嵌 JSX 手动控制显示文本。

**枚举类（状态）— 最简单：**
```tsx
// ❌ 错误 — SelectValue 空，依赖 placeholder/自动派生
<SelectTrigger id="status">
  <SelectValue />
</SelectTrigger>

// ✅ 正确 — JSX 内嵌显示文本
<SelectTrigger id="status">
  <SelectValue>{form.status === 0 ? "停用" : "启用"}</SelectValue>
</SelectTrigger>
<SelectContent>
  <SelectItem value="1">启用</SelectItem>
  <SelectItem value="0">停用</SelectItem>
</SelectContent>
```

**选项类（部门/角色）— 需要查找：**
```tsx
// value 传空字符串表示未选中
<Select
  value={form.deptId == null ? "" : String(form.deptId)}
  onValueChange={(v) => onFormChange({ deptId: v === "" ? null : Number(v) })}
>
  <SelectTrigger>
    <SelectValue>
      {form.deptId == null
        ? "请选择部门"
        : (deptOptions.find((o) => o.id === form.deptId)?.label ?? "请选择部门")}
    </SelectValue>
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="">无</SelectItem>
    {deptOptions.map((opt) => (
      <SelectItem key={opt.id} value={String(opt.id)}>{opt.label}</SelectItem>
    ))}
  </SelectContent>
</Select>
```

**数组类（角色单选）— 同理查找：**
```tsx
<Select
  value={form.roleIds && form.roleIds.length > 0 ? String(form.roleIds[0]) : ""}
  onValueChange={(v) => onFormChange({ roleIds: v === "" ? [] : [Number(v)] })}
>
  <SelectTrigger>
    <SelectValue>
      {form.roleIds && form.roleIds.length > 0
        ? (roles.find((r) => r.id === (form.roleIds ?? [])[0])?.roleName ?? "请选择角色")
        : "请选择角色（单选）"}
    </SelectValue>
  </SelectTrigger>
  <SelectContent>
    {roles.map((role) => (
      <SelectItem key={role.id} value={String(role.id)}>{role.roleName}</SelectItem>
    ))}
  </SelectContent>
</Select>
```

**关键要点：**
- `value=""`（空字符串）表示未选中，用 `""` 而非 `undefined`
- `SelectValue` 不用 `placeholder` 属性，改用 JSX children 手动渲染显示内容
- 显示文本通过 `find()` 从 options 数组中查找匹配项

---

## 11. 数据Table筛选下拉 — SelectItem value 设为中文文本（踩坑实录）

**问题：** 数据Table顶部的状态筛选下拉，`SelectValue` 显示数字而非中文。

**场景：** 流程实例（0=全部/1=运行中/2=已完成/3=已取消）、任务（0=全部/1=待处理/2=已完成）等筛选下拉。

**根因：** Base UI `SelectValue` 的显示逻辑是：用当前 `value` prop 值去匹配 `SelectItem value=`，匹配上就渲染对应 `SelectItem` 的 children 文本。如果 `SelectItem value="0">全部状态</SelectItem>` 且 `value={0}`（number），`SelectValue` 直接渲染字符串 `"0"` 而非 `"全部状态"`。`placeholder` 属性在有值时完全不生效。

**❌ 错误方案（已踩坑）：** 用 `Number(v)` 转中文值 → `Number("运行中")` = `NaN`，导致筛选 `item.status === NaN` 永远为 `false`，所有数据被过滤掉显示空白。

**✅ 正确方案：** 用字符串比较代替 `Number()`，且必须处理 `null`：
```tsx
// ❌ 错误
onValueChange={(v) => setStatusFilter(v === "全部状态" ? 0 : Number(v))}  // NaN!

// ✅ 正确 — 字符串比较，处理 null
onValueChange={(v) => setStatusFilter(
  v === null || v === "全部状态" ? 0
  : v === "运行中" ? 1
  : v === "已完成" ? 2
  : 3
)}
```

**状态映射（筛选下拉专用）：**
```typescript
const statusMap: Record<number, { label: string; color: string }> = {
  1: { label: "运行中", color: "text-blue-500" },
  2: { label: "已完成", color: "text-green-500" },
  3: { label: "已取消", color: "text-muted-foreground" },
}
// 筛选逻辑
const filtered = data.filter(item => statusFilter === 0 || item.status === statusFilter)
```

**与表单场景的区别：**
- 表单编辑：用 JSX children 精确控制显示文本（section 10）
- 筛选下拉：用中文 value + 字符串比较解析回数字状态码（推荐）

---

## 12. 前后端字段名不一致 — `realName` vs `nickname`

**问题：** 后端 `UserCreateDTO` / `UserUpdateDTO` 用 `realName`（@NotBlank），前端 `SysUser` 接口曾误用 `nickname`，导致表格真实姓名始终显示"-"。

**排查方法：** 创建测试数据后直接调 API 看返回字段名；或对比后端 DTO 和前端 interface 定义。

**monbo-bpm 项目字段对照：**
| 后端 DTO | 前端 SysUser | 说明 |
|----------|-------------|------|
| `realName` | `realName` | 真实姓名，@NotBlank |
| `username` | `username` | 用户名 |
| `email` | `email` | 邮箱 |
| `phone` | `phone` | 手机 |
| `status` | `status` | 1启用/0停用 |

**修复：** 前后端统一用 `realName`，表单字段和表格列同步修改。

---

## 关键文件
- `src/components/ui/dropdown-menu.tsx` - shadcn Base UI 封装
- `src/components/layout/nav-user.tsx` - button嵌套问题的实际解决方案
- `src/components/layout/app-sidebar.tsx` - SidebarMenuButton+`<a>`嵌套问题的解决方案 + 动态isActive
- `src/lib/api.ts` - extractData + 后端响应格式处理
- `vite.config.ts` - Vite 层 `@` 别名配置
