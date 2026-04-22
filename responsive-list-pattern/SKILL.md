---
name: responsive-list-pattern
description: PC/Mobile自适应列表组件 - 根据useIsMobile自动切换Table和Card视图
category: frontend
tags: [mobile, responsive, react, shadcn]
---

# ResponsiveList — PC/Mobile自适应列表组件

## Problem
每个列表页面（用户、角色、部门等）需要同时支持PC端表格和移动端卡片两种视图，手动写两套UI代码重复且难维护。

## Solution
通用`ResponsiveList<T>`组件，根据`useIsMobile()`自动切换：

- **PC端**：原生HTML表格，hover行高亮
- **Mobile端**：Card堆叠，卡片顶部显示ID+操作按钮，每行是label-value对

## Core Pattern

```tsx
import { useIsMobile } from '@/hooks/use-mobile';
import { cn } from '@/lib/utils';
import { Card, CardContent } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Edit, Trash2, Loader2 } from 'lucide-react';

interface Column<T> {
  key: string;
  label: string;
  render?: (row: T) => React.ReactNode;
  className?: string;
}

interface ResponsiveListProps<T> {
  columns: Column<T>[];
  data: T[];
  keyField: keyof T;
  onEdit?: (row: T) => void;
  onDelete?: (row: T) => void;
  deleteLoading?: number | null;
  emptyText?: string;
}

function ResponsiveList<T extends Record<string, any>>({
  columns, data, keyField, onEdit, onDelete, deleteLoading, emptyText = '暂无数据',
}: ResponsiveListProps<T>) {
  const isMobile = useIsMobile();

  if (isMobile) {
    return (
      <div className="flex flex-col gap-3">
        {data.length === 0 ? (
          <Card><CardContent className="py-8 text-center">{emptyText}</CardContent></Card>
        ) : data.map((row) => (
          <Card key={String(row[keyField])} className="break-inside-avoid">
            <CardHeader className="pb-2 px-4 pt-3">
              <div className="flex justify-between items-center">
                <span className="text-xs font-mono">#{row[keyField]}</span>
                <div className="flex gap-1">
                  {onEdit && <Button variant="ghost" size="icon" className="h-7 w-7" onClick={() => onEdit(row)}><Edit className="h-3.5 w-3.5"/></Button>}
                  {onDelete && <Button variant="ghost" size="icon" className="h-7 w-7 text-destructive" onClick={() => onDelete(row)} disabled={deleteLoading === row[keyField]}>
                    {deleteLoading === row[keyField] ? <Loader2 className="h-3.5 w-3.5 animate-spin"/> : <Trash2 className="h-3.5 w-3.5"/>}
                  </Button>}
                </div>
              </div>
            </CardHeader>
            <CardContent className="px-4 pb-3 pt-0 space-y-1.5">
              {columns.filter(c => c.key !== 'actions').map(col => (
                <div key={col.key} className="flex justify-between gap-2 text-xs">
                  <span className="text-muted-foreground shrink-0">{col.label}</span>
                  <span className="text-right font-medium truncate max-w-[60%]">
                    {col.render ? col.render(row) : String(row[col.key] ?? '-')}
                  </span>
                </div>
              ))}
            </CardContent>
          </Card>
        ))}
      </div>
    );
  }

  // Desktop: native HTML table (NOT shadcn Table — it doesn't exist in this project)
  return (
    <div className="rounded-md border overflow-hidden">
      <table className="w-full text-sm">
        <thead className="bg-muted/50 border-b">
          <tr>
            {columns.map(col => (
              <th key={col.key} className={cn('px-4 py-3 text-left font-medium text-xs uppercase tracking-wide text-muted-foreground', col.className)}>
                {col.label}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {data.length === 0 ? (
            <tr><td colSpan={columns.length} className="h-24 text-center">{emptyText}</td></tr>
          ) : data.map(row => (
            <tr key={String(row[keyField])} className="border-b hover:bg-muted/30 transition-colors">
              {columns.map(col => (
                <td key={col.key} className={cn('px-4 py-3', col.className)}>
                  {col.render ? col.render(row) : String(row[col.key] ?? '-')}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}

export { ResponsiveList };
export type { Column };
```

## Usage

```tsx
import { ResponsiveList } from '@/components/layout/ResponsiveList';

<ResponsiveList
  columns={[
    { key: 'id', label: 'ID', className: 'w-16' },
    { key: 'username', label: '用户名' },
    { key: 'email', label: '邮箱' },
    { key: 'status', label: '状态', render: (u) => <StatusBadge status={u.status} /> },
    {
      key: 'actions',
      label: '操作',
      render: (u) => (
        <div className="flex gap-2">
          <Button variant="ghost" size="sm" onClick={() => handleEdit(u)}><Pencil/></Button>
          <Button variant="ghost" size="sm" onClick={() => handleDelete(u)}><Trash2/></Button>
        </div>
      ),
    },
  ]}
  data={users}
  keyField="id"
  onEdit={handleEdit}
  onDelete={handleDelete}
  deleteLoading={deleteLoadingId}
  emptyText="暂无用户"
/>
```

## Pitfalls

1. **不要用`@/components/ui/table`** — shadcn/ui的Table组件在此项目中不存在，会报`Cannot find module '@/components/ui/table'`
2. **用原生HTML table + tailwind** — PC端桌面视图用原生`<table>`最可靠
3. **Mobile端Card的break-inside-avoid** — 防止卡片被截断
4. **deleteLoading比较** — 用`row[keyField]`（即id）比较，不要用index

## Files

- `/src/components/layout/ResponsiveList.tsx` — 组件实现
- `/src/pages/users/UserListPage.tsx` — 接入示例
- `/src/pages/roles/RoleListPage.tsx` — 接入示例
