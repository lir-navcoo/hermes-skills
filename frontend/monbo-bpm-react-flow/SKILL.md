---
name: monbo-bpm-react-flow
description: monbo-bpm React Flow implementation patterns, node types, and gotchas
---

# monbo-bpm React Flow Implementation

## 项目信息
- 项目路径：`~/monbo-bpm/monbo-bpm-ui/`
- 技术栈：React Flow + shadcn/ui + Tailwind CSS + TypeScript (verbatimModuleSyntax)
- 包管理：npm

## 依赖安装
```bash
npm install reactflow
# 卸载旧依赖
npm uninstall @xrenders/xflow
```

## React Flow 关键实践

### 1. TypeScript verbatimModuleSyntax 下必须用 import type
所有类型导入必须分开：
```ts
import { Handle, Position } from 'reactflow'
import type { NodeProps } from 'reactflow'  // 必须单独写
```

### 2. fitView 必须限制 maxZoom
`fitView` 在内容少时会自动放大（可达1.5x），必须限制：
```tsx
const handleFitView = useCallback(
  () => reactFlowInstance.fitView({ padding: 0.2, maxZoom: 1 }),
  [reactFlowInstance]
)
```

### 3. 自定义节点容器与元素的垂直对齐
当节点使用「容器 div + 边缘 handles」结构时，**不要手动计算高度和偏移**——用 flex 布局让元素自然撑开，完全避免高度不一致的视觉 bug。

推荐方案（flex 布局）：
```tsx
// ✅ 用 flex 替代绝对定位 + 手动高度计算
<div className="relative flex flex-col items-center" style={{ width: 80 }}>
  <Handle type="target" position={Position.Top} style={{ top: -5, left: '50%' }} />
  {/* card 自然撑开高度，不设固定 height */}
  <div className="node-card flex items-center justify-center gap-1.5 rounded-xl border-2 ..."
       style={{ width: 80, minHeight: 48, padding: '8px 12px' }}>
    <Icon style={{ width: 16, height: 16, flexShrink: 0 }} />
    <span>...</span>
  </div>
  {/* handles 用绝对定位挂在容器边缘 */}
  {branches.map((_, i) => (
    <Handle type="source" position={Position.Bottom}
            style={{ bottom: -5, left: `${((i+1)/(branches.length+1))*100}%` }} />
  ))}
</div>
```

❌ 错误方案（手动算高度偏移）：
```tsx
// ❌ 容器设死高度，card 用 top: 0 贴顶，导致 card 和 handles 之间有巨大空白
style={{ width: 80, height: 48 + branchCount * 24 }}
style={{ top: 0 }}

// ❌ 手动居中——计算容易出错，内容撑开时高度对不上
style={{ top: (totalHeight - cardHeight) / 2 }}
```

### 4. Handle 必须有显式 id（关键！）
所有 Handle 必须写显式 `id`，否则 React Flow 自动生成的 id 不可预测，导致 `onConnect` 中 `connection.sourceHandle` 解析分支索引时出现 `NaN`。

```tsx
// ✅ 正确：每个 Handle 有明确 id
<Handle type="target" position={Position.Top} id="target" />
<Handle type="source" position={Position.Bottom} id="source" />
// 网关的多分支 source 用 out-0, out-1, out-2...
<Handle type="source" position={Position.Bottom} id={`out-${i}`} />

// ❌ 错误：没有 id，React Flow 会自动生成，格式不可控
<Handle type="source" position={Position.Bottom} />
```

### 5. onConnect 中解析网关分支索引
`connection.sourceHandle` 格式为 `"out-{branchIndex}"`（如 `"out-0"`），解析方式：
```ts
const branchIndex = parseInt(connection.sourceHandle!.replace('out-', ''), 10)
```
在 `connection.sourceHandle` 为 `null`（非网关节点连线）时需要判空，避免 `parseInt(null.replace(...))` 报错。

### 6. Lucide Icon 名称注意
- `CallIcon` / `FolderInput` → 不存在，用 `FileInput`
- `GitFork` = 排他网关，`GitMerge` = 并行网关，`GitBranch` = 包容网关

### 7. React Flow CSS 清理
```css
.react-flow__attribution { display: none !important; }
```

## 节点类型注册
```ts
// nodes/index.tsx
import StartEndNode from './StartEndNode'
// ... 其他节点

export const nodeTypes = {
  START: StartEndNode,
  END: StartEndNode,  // 共用同一组件，通过 data.nodeType 区分
  USER_TASK: UserTaskNode,
  // ...
}
```

## Edge 类型注册
```ts
import BpmnEdge from './BpmnEdge'
export const edgeTypes = { bpmn: BpmnEdge }
```

### 8. 全局模块状态与 Vite HMR
流程设计器使用模块级 `nodeIdCounter` 生成唯一节点 ID。Vite HMR 不会重置模块级变量，导致热更新后计数器"记忆"旧值，产生 ID 碰撞。

必须用 `import.meta.hot` 显式清理：
```ts
let nodeIdCounter = 1
const getId = () => `node_${Date.now()}_${nodeIdCounter++}`

// Vite HMR 时重置计数器
if (import.meta.hot) {
  import.meta.hot.dispose(() => {
    nodeIdCounter = 1
  })
}
```

### 9. BPMN 合法性校验模式
流程保存/导出前应做合法性校验，至少检查：
- 存在且仅有一个开始节点、一个结束节点
- 所有边的 source/target 指向存在的节点
- 除开始节点外，所有节点都有入边和出边
- 节点 ID 全局唯一（导入时校验）

```ts
interface ValidationResult { valid: boolean; errors: string[] }

function validateBpmn(nodes: Node[], edges: Edge[]): ValidationResult {
  // ...
}
```

## 构建验证
```bash
npm run build  # tsc -b && vite build
```

## 开发启动
```bash
rm -rf node_modules/.vite  # 清除过期的 Vite 优化缓存
npm run dev
```
