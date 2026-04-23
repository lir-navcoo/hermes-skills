---
name: tanstack-table-tree-expand
description: TanStack Table 树形表格实现踩坑记录 — 展开/收起 + Base UI DropdownMenuTrigger 注意事项
category: frontend
---

# TanStack Table 树形表格展开/收起实现

## 核心数据结构

API 返回嵌套树形结构（已完整，无需 buildTree）：
```json
[
  {
    "id": 1,
    "deptName": "总公司",
    "children": [
      { "id": 2, "deptName": "研发部", "children": [...] }
    ]
  }
]
```

## 关键：expandedIds 必须传入 flattenTree

`flattenTree` 的递归必须接收 `expandedIds` Set，才能只输出当前展开的节点：

```tsx
function flattenTree(tree: Department[], expandedIds: Set<number>, level = 0): FlatNode[] {
  const result: FlatNode[] = []
  for (const dept of tree) {
    const children = dept.children ?? []
    result.push({ dept, level, hasChildren: children.length > 0 })
    // 只有当前节点在 expandedIds 中，才继续递归子节点
    if (children.length > 0 && expandedIds.has(dept.id)) {
      result.push(...flattenTree(children, expandedIds, level + 1))
    }
  }
  return result
}
```

**错误做法：** 先 flatten 所有节点，再用 `expandedIds` 控制样式 → 收起时行仍然存在。

## API 已是树形时，不要 buildTree

API 返回树形数据（children 字段已嵌套），只需要排序，不需要重建树：
```tsx
function sortTree(tree: Department[]): Department[] {
  const sorted = [...tree].sort((a, b) => (a.sortOrder ?? 0) - (b.sortOrder ?? 0))
  sorted.forEach(dept => {
    if (dept.children?.length > 0) dept.children = sortTree(dept.children)
  })
  return sorted
}
```

## Base UI DropdownMenuTrigger 注意

shadcn v4 使用 `@base-ui/react`，`DropdownMenuTrigger`：
- **不存在 `asChild` prop**（报错：`Property 'asChild' does not exist`）
- **不存在 `class` prop**（报错：`Property 'class' does not exist on IntrinsicAttributes & Props<unknown>`）
- `className` 是正确 prop

正确写法：
```tsx
<DropdownMenuTrigger className="inline-flex items-center justify-center w-8 h-8 rounded text-muted-foreground hover:bg-accent">
  <IconDotsVertical className="size-4" />
</DropdownMenuTrigger>
```

## 展开列宽度

展开图标按钮用 `w-6 h-6`（24px），placeholder span 用 `w-6`。
