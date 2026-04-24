---
name: macos-tcc-permissions
description: macOS TCC沙盒权限限制 — Hermes Agent无法通过execute_code/delegate_task/browser访问本地文件，即使终端有Full Disk Access。解决方案：BOSS手动授权或粘贴输出。
category: devops
tags: [macos, tcc, permissions, sandbox]
---

# macOS TCC 权限限制

## 问题现象

execute_code / delegate_task / browser 访问本地文件全部 PermissionError: [Errno 1] Operation not permitted，即使终端.app已开启Full Disk Access。

## 根因

macOS TCC (Transparency, Consent and Control) 按进程沙盒授权。终端.app有权限，但Hermes agent运行在独立进程/沙盒中，没有Documents文件夹的TCC授权。

## 解决方案

### 方案1：TCC手动授权（推荐）
**System Settings → Privacy & Security → Files and Folders** → 找到Hermes条目 → 勾选Documents权限

注意：授权的是具体进程的Bundle ID，需确认是哪个进程在承载agent

### 方案2：BOSS粘贴输出
对于简单文件读取，让BOSS在终端执行 `cat` 或 `ls` 后粘贴输出

### 方案3：SSH到有权限的机器
如果文件在远程服务器，通过SSH执行命令

## 不可行方案

- `file://` 浏览器协议 → 返回空页面
- delegate_task 子进程 → 同样沙盒限制
- Python os.listdir / pathlib → PermissionError
- subprocess.run(['ls', ...]) → PermissionError

## 预防

新建项目时明确告知BOSS：Hermes沙盒可能无法直接读写 ~/Documents 等受保护目录，必要时需BOSS协助或手动授权。
