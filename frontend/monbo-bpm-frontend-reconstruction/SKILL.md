---
name: monbo-bpm-frontend-reconstruction
description: monbo-bpm 前端重构进度记录 - 移动端+多主题+中英双语
---

# monbo-bpm 前端重构记录

## 重构目标
- 移动端支持（ResponsiveList 组件）
- 多主题（暗色模式）
- 中英双语（i18next）

## 当前进度（2026-04-24）

### 已完成
- ThemeProvider + dark mode ✅
- i18next 配置 + 翻译文件（locales/zh.json, locales/en.json）✅
- Header 右侧工具栏（主题切换 + 语言切换）✅
- 6 个列表页改 ResponsiveList ✅
  - users / roles / departments / process-defs / process-insts / tasks

### 待处理
- 修 build TS 错误（ThemeProvider children类型、asChild→render prop）
- 修设计器 bug：删除节点后添加新节点默认 label 显示"结束"
- 流程实例列表页面重建
- MyTasks 页面重建
- 移动端侧边栏（Drawer 替换 Sheet）
- 中英翻译文本覆盖（当前 6 个列表页已用 t()，其他页面待补）

## 技术栈
- shadcn v4（Base UI，render prop 替代 asChild）
- Tailwind CSS v4
- i18next + react-i18next
- TanStack Table + ResponsiveList 组件
- 路径：monbo-bpm-ui/monbo-bpm-ui/

## Build 状态
- 有 TS 错误待修，build 未通过
