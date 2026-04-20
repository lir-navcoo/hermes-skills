---
name: github-api-push-with-build
description: GitHub 推送 + 本地 build 验证工作流，适用于 GitHub Pages 前端项目
category: github
---

# GitHub 推送 + Build 验证工作流

## 何时用
向 GitHub 仓库推送代码时（尤其是前端项目）。

## 标准流程

### 1. 本地修改文件

### 2. 本地验证 build
```bash
npm install
npm run build
```
确认 TypeScript 编译 + Vite 构建均成功后再推送。

### 3. 推送（两种方式）

**方式 A：git remote set-token（推荐）**
```bash
git remote set-url origin https://TOKEN@github.com/user/repo.git
git add . && git commit -m "message" && git push
```
注意：TOKEN 替换为真实 GitHub PAT。

**方式 B：GitHub API（git push 失败时用）**
```bash
SHA=$(gh api "repos/user/repo/contents/path?ref=main" --jq '.sha')
CONTENT=$(base64 -i path/to/file)
gh api --method PUT "repos/user/repo/contents/path" \
  -f message="commit message" \
  -f content="$CONTENT" \
  -f sha="$SHA"
```

### 4. 等待 GitHub Actions
```bash
gh api repos/user/repo/actions/runs --jq '.workflow_runs[0] | {status, conclusion}'
```
等待 `conclusion` 变为 `success` 或 `failure`。

## 常见错误

### TypeScript `as const` 字面量类型不兼容（高频踩坑）
**错误**：`Labels` 类型用 `typeof labels` 时，中英文对象字面量 TS 报错「类型不可分配」。
**根因**：TypeScript 推断两个 `as const` 对象为不同字面量类型，即使结构完全一致。
**解决**：
```typescript
// ❌ 报错：Type '"Name"' is not assignable to type '"姓名"'
type Labels = typeof labels

// ✅ 正确：使用 Record<string, string>
type Labels = Record<string, string>
const i18n = (lang: 'zh' | 'en'): Labels => lang === 'en' ? labelsEn : labels
```

### git push / git remote set-url 均报 "Device not configured"
**原因**：git-credential helper 在本环境不可用，任何走 git credential 的方式都失败。
**解决**：始终用 GitHub API 方式（方式 B）推送，不尝试 git push。
> 教训（高频踩坑）：本环境 git credential 永远失败，任何 git push 方式都不可用。必须本地 build 验证 → GitHub API 推送 → 等待 Actions 确认。

### 简历站特例（/tmp/resume-build）
```bash
# 首次：克隆（--depth=1 加速）
git clone --depth=1 https://github.com/lir-navcoo/resume.git /tmp/resume-build

# 修改后：复制文件 + 本地验证 build
cp /tmp/resume_app.tsx /tmp/resume-build/src/App.tsx
cd /tmp/resume-build && npm install && npm run build

# 确认 build 成功（无 TS 错误）后，用 GitHub API 推送
SHA=$(gh api "repos/lir-navcoo/resume/contents/src/App.tsx?ref=main" --jq '.sha')
CONTENT=$(base64 -i /tmp/resume_app.tsx)
gh api --method PUT "repos/lir-navcoo/resume/contents/src/App.tsx" \
  -f message="commit message" -f content="$CONTENT" -f sha="$SHA"

# 推送后轮询 Actions 状态（每次 push 后必须执行）
sleep 35 && gh api repos/lir-navcoo/resume/actions/runs --jq '.workflow_runs[0] | {status, conclusion}'
```

### 简历站新增数据条目字段规范
在 `resumeData` 中新增可选字段（如 `achievement`）时：
1. 直接在对应条目对象中添加 `achievement: '内容'`
2. JSX 渲染处使用 `(exp as any).achievement` 访问（避免修改全局类型）
3. 条件渲染用 `{(exp as any).achievement && (...)}`

### TypeScript `as const` 字面量类型不兼容（高频踩坑）
**错误**：`Labels` 类型用 `typeof labels` 时，中英文对象字面量 TS 报错「类型不可分配」。
**根因**：TypeScript 推断两个 `as const` 对象为不同字面量类型，即使结构完全一致。
**解决**：
```typescript
// ❌ 报错：Type '"Name"' is not assignable to type '"姓名"'
type Labels = typeof labels

// ✅ 正确：使用 Record<string, string>
type Labels = Record<string, string>
const i18n = (lang: 'zh' | 'en'): Labels => lang === 'en' ? labelsEn : labels
```

### TypeScript 可选字段属性不存在（新增字段场景，高频踩坑）
**错误**：`Property 'achievement' does not exist on type '...'` — 给数据条目新增可选字段时，已定义的类型没有该字段。
**解决**：在 JSX 中使用类型断言 `(exp as any).achievement`，避免修改全局数据类型。
```typescript
// ✅ 在 JSX 渲染中
{(exp as any).achievement && (
  <div>{(exp as any).achievement}</div>
)}
```

### GitHub Actions build 失败排查（高频踩坑）
**问题**：GitHub Actions 日志不对外暴露 API，无法远程排查失败原因。
**解决**：克隆仓库到本地完整 build 验证。
```bash
git clone --depth=1 https://github.com/user/repo.git /tmp/repo-build
cd /tmp/repo-build && npm install && npm run build
# 本地 build 失败时，错误信息直接输出在终端
```
