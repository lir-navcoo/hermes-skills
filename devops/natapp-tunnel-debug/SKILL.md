---
name: natapp-tunnel-debug
description: Natapp 内网穿透调试踩坑记录 - macOS SSH + Natapp 连接问题排查
tags: [natapp, tunnel, ssh, macos, debugging]
date: 2026-04-24
---

# Natapp Tunnel Debug 踩坑记录

## 问题场景
用 Natapp 将本地端口（SSH 22）暴露到公网，让其他人能通过 SSH 连接本机。

## 常见错误排查

### 1. ENOTFOUND https://navcoo.natapp4.cc
- **原因**：地址带了 `https://` 前缀
- **解决**：Natapp 是 TCP 隧道，直接用域名，不带协议头
  ```bash
  ssh -p 22 用户名@navcoo.natapp4.cc
  ```

### 2. ECONNREFUSED 118.31.62.7:22
- **可能原因**：
  1. Natapp 服务器宕机/维护
  2. Natapp 隧道未正确建立
  3. **本地 SSH 服务器未开启** ← 高频踩坑点
  4. 免费版 Natapp 不支持 TCP 端口转发

## 关键排查步骤

```bash
# 查看 Natapp 是否在运行
ps aux | grep natapp | grep -v grep

# 查看 Natapp 网络连接（应该有两个：443 和 4443）
lsof -i -P | grep natapp

# 查看本地 SSH 状态
sudo systemsetup -getremotelogin
# 输出 "Remote Login: Off" = SSH 未开启

# 查看本地 22 端口是否监听
sudo lsof -i :22 -n -P
# 应该有 launchd 在 *:22 LISTEN

# 开启 macOS SSH
sudo systemsetup -setremotelogin on
```

## Natapp 配置文件的正确格式

Natapp CLI 不支持 `-remote_addr` 参数（flag provided but not defined）。必须用配置文件：

```ini
[common]
authtoken=你的token
loglevel=INFO
log=stdout
```

注意：`[client]` 段落可能免费版不支持。

## 根本原因

**Natapp 免费版/基础版可能不支持 TCP 反向隧道（端口映射到本地 SSH）**

nc 能连通 22 端口只说明 Natapp 服务器的 22 端口有响应，但不等于流量转发到了本地。

## 推荐方案：frp

如果需要可靠的 TCP 端口转发，换用 frp：

**服务端（公网服务器）：**
```toml
# frps.toml
[common]
bind_port = 7000
bind_addr = 0.0.0.0
```

**客户端（本地 Mac）：**
```toml
# frpc.toml
[common]
server_addr = 服务器IP
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
```

别人连接：`ssh 用户名@服务器IP -p 6000`

frp 完全自控，无第三方限制。
