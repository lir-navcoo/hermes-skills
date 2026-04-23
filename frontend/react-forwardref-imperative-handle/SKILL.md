---
name: react-forwardref-imperative-handle
description: React forwardRef + useImperativeHandle 最佳实践，避免常见陷阱
trigger: 当需要用 ref 调用子组件方法时加载
---

# React forwardRef + useImperativeHandle 最佳实践

## 基本模式

```typescript
import { forwardRef, useImperativeHandle, useCallback } from 'react'

interface MyComponentProps {
  onSave?: (data: any) => void
}

interface ImperativeRef {
  triggerSave: () => void
  triggerExport: () => void
  triggerReset: () => void
}

// 正确：组件用 forwardRef，类型两个参数（ref类型, props类型）
const MyComponent = forwardRef<ImperativeRef, MyComponentProps>((props, ref) => {
  // 1. 先定义所有 handler
  const handleSave = useCallback(() => {
    props.onSave?.()
  }, [props.onSave])

  const handleExport = useCallback(() => {
    // ...
  }, [])

  const handleReset = useCallback(() => {
    // ...
  }, [])

  // 2. useImperativeHandle 放在所有 handler 定义之后
  useImperativeHandle(ref, () => ({
    triggerSave: handleSave,
    triggerExport: handleExport,
    triggerReset: handleReset,
  }), [handleSave, handleExport, handleReset])

  // 3. 最后才 return JSX
  return <div>...</div>
})

MyComponent.displayName = 'MyComponent'
export default MyComponent
```

## ⚠️ 关键陷阱

### 陷阱1：useImperativeHandle 必须在 handler 定义之后
**错误顺序（会导致 ref 方法 undefined）**：
```typescript
const MyComponent = forwardRef<ImperativeRef, Props>((props, ref) => {
  // ❌ useImperativeHandle 太靠前
  useImperativeHandle(ref, () => ({ triggerSave: handleSave }), [handleSave])

  // handler 定义在 useImperativeHandle 之后
  const handleSave = useCallback(() => { ... }, [])
  const handleExport = useCallback(() => { ... }, [])

  return <div />
})
```

**正确顺序**：
```typescript
const MyComponent = forwardRef<ImperativeRef, Props>((props, ref) => {
  // 1. 所有 handler 先定义
  const handleSave = useCallback(() => { ... }, [])
  const handleExport = useCallback(() => { ... }, [])
  const handleReset = useCallback(() => { ... }, [])

  // 2. useImperativeHandle 放最后
  useImperativeHandle(ref, () => ({
    triggerSave: handleSave,
    triggerExport: handleExport,
    triggerReset: handleReset,
  }), [handleSave, handleExport, handleReset])

  return <div />
})
```

### 陷阱2：forwardRef 泛型参数顺序
```typescript
// ✅ 正确：forwardRef<Ref类型, Props类型>
const MyComponent = forwardRef<ImperativeRef, MyComponentProps>(...)

// ❌ 错误：顺序反了
const MyComponent = forwardRef<MyComponentProps, ImperativeRef>(...)
```

### 陷阱3：imperativeRef prop 类型不匹配
子组件暴露的 ref 类型必须与父组件 useRef 的类型一致：

```typescript
// 子组件
interface ImperativeRef {
  triggerSave: () => void
}

// 父组件使用时
const ref = useRef<ImperativeRef>(null)
designerRef.current?.triggerSave() // ✅

// 如果 ref 类型声明少了方法，TypeScript 会在调用时报错
```

### 陷阱4：useImperativeHandle deps 遗漏
每次 handler 变化都需要更新 deps，否则 ref 拿到的是旧引用：

```typescript
// ✅ 完整 deps
useImperativeHandle(ref, () => ({
  triggerSave: handleSave,
  triggerExport: handleExport,
}), [handleSave, handleExport]) // 两个都包含

// ❌ 遗漏导致 bug
useImperativeHandle(ref, () => ({
  triggerSave: handleSave,
  triggerExport: handleExport,
}), [handleSave]) // handleExport 变了但没更新
```

## 实际应用案例

### ProcessDesigner（monbo-bpm 项目）
```typescript
interface ProcessDesignerProps {
  initialData?: ProcessDefinition
  onSave?: (data: ProcessDefinition) => void
  imperativeRef?: RefObject<ImperativeRef>
  hideToolbar?: boolean
}

const ProcessDesigner = forwardRef<ImperativeRef, ProcessDesignerProps>((props, ref) => {
  // ... handlers ...

  const handleSave = useCallback(() => { ... }, [nodes, edges, props.onSave])
  const handleExport = useCallback(() => { ... }, [nodes, edges])
  // ... other handlers ...

  useImperativeHandle(props.imperativeRef, () => ({
    triggerSave: handleSave,
    triggerExport: handleExport,
    // ... all 8 methods
  }), [handleSave, handleExport, /* ... all handler deps ... */])

  return <div className="h-screen">...</div>
})
```

父组件调用：
```typescript
const ref = useRef<ImperativeRef>(null)
;<ProcessDesigner ref={ref} hideToolbar onSave={...} />

// 调用
ref.current?.triggerSave()
```

## 验证方法

在组件 return 之前加一行确认 ref 方法存在：
```typescript
console.log('ref methods:', {
  triggerSave: typeof ref?.current?.triggerSave,
  triggerExport: typeof ref?.current?.triggerExport,
})
```
