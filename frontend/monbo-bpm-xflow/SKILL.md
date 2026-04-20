---
name: monbo-bpm-xflow
description: Xflow BPMN流程设计器在monbo-bpm项目中的配置与踩坑记录
category: frontend
tags: [xflow, bpmn, react, monbo-bpm]
---

# monbo-bpm Xflow 流程设计器技术笔记

## Xflow 关键配置

### 初始缩放问题（200% zoom）
Xflow 内部 `es/XFlow.js`（ES模块/开发模式）有 `fitView: true`，导致 ReactFlow 自动计算缩放填充视口，可能产生 > 100% 的初始缩放。

**注意**：`lib/XFlow.js`（CommonJS/生产模式）已是 `fitView: false`，但 `es/XFlow.js` 不是。

**修复方案**：patch node_modules 的 es 目录

## postinstall 自动 Patch 脚本

Xflow 有多个需要 patch 的硬编码值，且 **必须同时 patch `lib/` 和 `es/` 两个目录**。

**scripts/patch-xflow.js**（ESM）：
```js
import fs from 'fs'
import path from 'path'
import { fileURLToPath } from 'url'
## patch-xflow.js（完整脚本）
需要 patch 4 类问题，涉及 6 个文件（lib/ + es/ 各一份）：

```js
import fs from 'fs'
import path from 'path'
import { fileURLToPath } from 'url'

const __dirname = path.dirname(fileURLToPath(import.meta.url))
const xflowRoot = path.join(__dirname, '../node_modules/@xrenders/xflow')

// XFlow.js — zoom 和 fitView 修复
const xflowTargets = [
  path.join(xflowRoot, 'lib/XFlow.js'),
  path.join(xflowRoot, 'es/XFlow.js'),
]

// NodeContainer — bgColor 修复
const nodeContainerTargets = [
  path.join(xflowRoot, 'lib/components/NodeContainer/index.js'),
  path.join(xflowRoot, 'es/components/NodeContainer/index.js'),
]

// === XFlow.js patches ===
for (const filePath of xflowTargets) {
  let content = fs.readFileSync(filePath, 'utf-8')
  let patched = false

  // Patch 1: fitView: true -> fitView: false, add maxZoom
  if (content.includes('minZoom: 0.3,') && content.includes('fitView: true')) {
    content = content.replace(
      /minZoom: 0\.3,\n(\s*)fitView: true,/,
      'minZoom: 0.3,\n    maxZoom: 1,\n$1fitView: false,'
    )
    patched = true
  }

  // Patch 2: fitViewOptions add maxZoom: 1
  if (content.includes('padding: 0.4') && !content.includes('maxZoom: 1,\n    },')) {
    content = content.replace(
      /padding: 0\.4,\n(\s*)\},/,
      'padding: 0.4,\n$1maxZoom: 1,\n$1},'
    )
    patched = true
  }

  if (patched) {
    fs.writeFileSync(filePath, content)
    console.log(`Patched: ${filePath}`)
  }
}

// === NodeContainer patches (bgColor fix) ===
for (const filePath of nodeContainerTargets) {
  let content = fs.readFileSync(filePath, 'utf-8')
  let patched = false

  if (content.includes('Object.assign({}, icon)')) {
    const patterns = [
      /_react\.default\.createElement\(IconBox, Object\.assign\(\{\}, icon\)\)/g,
      /React\.createElement\(IconBox, Object\.assign\(\{\}, icon\)\)/g,
    ]
    for (const pattern of patterns) {
      if (content.match(pattern)) {
        const replacement = pattern.source.includes('_react.default')
          ? '_react.default.createElement(IconBox, { type: icon.type, bgColor: undefined })'
          : 'React.createElement(IconBox, { type: icon.type, bgColor: undefined })'
        content = content.replace(pattern, replacement)
        patched = true
        break
      }
    }
  }

  if (patched) {
    fs.writeFileSync(filePath, content)
    console.log(`Patched: ${filePath}`)
  }
}
```

**package.json 配置**：
```json
"postinstall": "node scripts/patch-xflow.js"
  if (patched) {
    fs.writeFileSync(filePath, content)
    console.log(`Patched: ${filePath}`)
    totalPatched++
  } else {
    console.log(`Already patched or not needed: ${filePath}`)
  }
}

console.log(totalPatched > 0 ? `Done. ${totalPatched} file(s) patched.` : 'No patches applied.')
```

**package.json 配置**：
```json
"postinstall": "node scripts/patch-xflow.js"
```

### 排查记录
- 症状：初始化显示 200% 缩放
- 原因：`es/XFlow.js` 有 `fitView: true`，ReactFlow 自动计算填充视口
- `lib/XFlow.js` 和 `es/XFlow.js` 配置不同，需分别检查
- `bgColor` React 警告来自 Xflow 内部 `NodeContainer`，不影响功能，可忽略

## Xflow 已知限制

### 图标问题
Xflow 内部使用阿里云 iconfont CDN（`//at.alicdn.com/t/a/font_4069358_dd524fgnynb.js`）渲染节点图标。
- 网络访问失败时图标不显示
- 无法替换为本地 lucide-react 图标（架构绑定）
- 如需完全自定义图标，需换用 @xyflow/react 重写节点组件

### 缩放限制（重要）
Xflow 的 `fitView` 会自动计算缩放填充视口，可能产生 > 100% 的初始缩放。
必须通过 patch ReactFlow 的 `maxZoom` 和 `fitViewOptions.maxZoom` 来限制。Xflow 自身配置项中没有 maxZoom 选项。

**关键**：xflow 有两个构建目录，**都需要 patch**：
- `lib/` — CommonJS，生产构建用（vite build）
- `es/` — ES Modules，开发模式用（vite dev server）

zoom 相关 patch 需要同时应用到两个目录。

### bgColor React 警告修复
**问题**：`React does not recognize the bgColor prop on a DOM element`

**根因**：NodeContainer 中 `Object.assign({}, icon)` 把整个 icon 对象（含 bgColor）传给了 IconBox（@ant-design/icons 的 createFromIconfontCN），IconBox 内部把 `restProps`（含 bgColor）spread 到 `<span>` DOM 元素上。

**修复**：不 spread icon 对象，显式传参并设置 `bgColor: undefined`：
```js
// 错误：Object.assign({}, icon) 传递了 bgColor
React.createElement(IconBox, Object.assign({}, icon))

// 正确：只传必要字段
React.createElement(IconBox, { type: icon.type, bgColor: undefined })
```

需要 patch 的文件（两处，lib/ 和 es/）：
- `node_modules/@xrenders/xflow/lib/components/NodeContainer/index.js`
- `node_modules/@xrenders/xflow/es/components/NodeContainer/index.js`

### 全局变量 polyfill
Xflow UMD 构建依赖 Node.js 的 `global`，浏览器环境需要 polyfill。

**vite.config.ts 配置**：
```ts
define: {
  global: 'globalThis',
}
```

### 隐藏注释功能
通过 `globalConfig.controls.hideAnnotate = true` 隐藏。

### Xflow 配置结构
```tsx
<XFlow
  settings={nodeSettings}
  initialValues={initialValues}
  layout="TB"
  globalConfig={{
    controls: { 
      hideAnnotate: true,
      hideAutoLayout: true,
      hideZoomInOutBtns: true,
    },
  }}
/>
```

### TControl 可配置项
```ts
interface TControl {
  hideAddNode?: boolean;
  hideAnnotate?: boolean;      // 隐藏注释
  hideUndoRedoBtns?: boolean;  // 隐藏撤销/重做
  hideZoomInOutBtns?: boolean; // 隐藏缩放按钮
  hideControlBtns?: boolean;   // 隐藏整个控制栏
  hideAutoLayout?: boolean;     // 隐藏自动布局
  hideFullscreen?: boolean;
  hideInteractionMode?: boolean;
}
```

## 节点配置 (nodeSettings)

### icon 配置
```ts
icon: {
  type: 'icon-start',  // iconfont 图标名称
  bgColor: '#17B26A',
}
```

**注意**：默认使用阿里云 CDN iconfont，可能在国内网络环境下加载失败，导致图标不显示。
如需替换，可通过 `iconFontUrl` 属性传入自定义 iconfont URL。

### hidden 节点
`hidden: true` 表示在左侧节点菜单中隐藏该节点类型，但画布上仍可拖拽添加。

### 必填字段 (TNodeItem)
- `switchExtra`: 条件分支配置
- `parallelExtra`: 并行分支配置
- `onTesting`: 测试回调函数
- `settingSchema.required`: 必须声明为 `string[]` 类型断言，避免 TS 报错
