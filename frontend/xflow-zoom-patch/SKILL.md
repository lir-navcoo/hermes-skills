---
name: xflow-zoom-patch
description: Fix @xrenders/xflow initial zoom (80% hardcoded) via postinstall patch
---

# Xflow Initial Zoom Patch

## Problem
@xrenders/xflow 内部 `zoomTo(0.8)` 硬编码初始缩放为 80%，无法通过 props 配置。

## Solution
npm install 后自动 patch：

**package.json scripts:**
```json
"postinstall": "sed -i 's/zoomTo(0.8)/zoomTo(1)/g' node_modules/@xrenders/xflow/lib/XFlow.js"
```

**vite.config.ts define (for global polyfill):**
```ts
define: {
  global: 'globalThis',
}
```

## Why
- Xflow 是 UMD 包，依赖 Node.js `global` 变量，浏览器需要 polyfill
- 初始缩放在 useEffect 里写死，无法外部覆盖
- postinstall 确保每次 npm install 后自动修复
