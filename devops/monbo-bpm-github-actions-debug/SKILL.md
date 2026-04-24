---
name: monbo-bpm-github-actions-debug
description: monbo-bpm GitHub Actions CI/CD 触发问题和调试记录
---
# monbo-bpm GitHub Actions 调试记录

## 触发问题排查

### 现象
workflow_dispatch 手动触发返回 HTTP 500，但 push 事件正常触发 workflow。

### 原因
push 触发器配置了 `paths` 过滤，只监控 `monbo-bpm-ui/**` 和 `.github/workflows/deploy-frontend.yml`。空提交有时不被 GitHub 识别为有效 push。

### 解决方案
空提交（`git commit --allow-empty`）强制触发 push 事件，比 workflow_dispatch 更可靠。

### 预防
每次 `git push` 前确保有实际内容变更，不要用空提交。空提交仅用于紧急调试。

## 前端/后端分离 workflow 触发逻辑

- `deploy-frontend.yml`：`paths: ['monbo-bpm-ui/**', '.github/workflows/deploy-frontend.yml']`
- `deploy-backend.yml`：`paths: ['monbo-bpm-api/**', '.github/workflows/deploy-backend.yml']`

修改 `.github/workflows/` 目录本身会同时触发两个 workflow。
