---
name: server-deployment-troubleshoot
description: CentOS Stream 9 SSH 故障排查与部署技巧
tags: [centos, ssh, dnf, deployment, troubleshooting]
---

# Server Deployment Troubleshoot

## CentOS Stream 9 Server SSH 故障排查

### 症状
SSH 连接（端口 22）在执行 `dnf install` 大型包（java, mysql, nginx, openssh-server）时开始被拒绝：
```
kex_exchange_identification: Connection closed by remote host
```
但 `nc -zv <IP> 22` 显示端口仍通。

### 根因
dnf 安装/升级 openssh-server 后，sshd 服务可能处于不稳定状态或正在重启，sshd 在 key exchange 阶段就关闭连接。

### 关键发现
**备选 SSH 端口**：在主端口 22 被阻塞时，端口 **2222** 可能开着（云厂商的跳板机端口）。验证命令：
```bash
nc -zv -w 5 <IP> 2222
```

### 正确做法
1. 先 `nc -zv` 检查所有端口 22/2222 是否通
2. 如果 22 被拒但 2222 通，说明 sshd 阻塞/不稳定
3. 通过云控制台 VNC 登录，重启 sshd：`systemctl restart sshd`
4. 或等几分钟让 dnf 安装完成（后台运行 `nohup dnf install ... &`）

### 预防措施
- 大型 dnf 安装用 systemd oneshot 服务托管，不依赖 SSH 保持连接
- 分步安装：先装基础包确认 SSH 不受影响，再装重型包

### 排查工具对比（2026-04-21 实测）

| 工具 | 可用性 | 效果 |
|------|--------|------|
| `nc -zv <IP> 22` | 系统自带 | TCP层连通性验证，最快最准 |
| `expect` | `/usr/bin/expect` | 可自动化密码+多命令，但交互超时不稳定 |
| `paramiko` (Python) | 需`pip3 install paramiko` | 最可靠但sandbox环境可能无此模块 |
| `sshpass` | 需安装 | macOS默认无此命令 |

### SSH连接失败时的辅助排查
即使SSH连不上，也可以探测应用层端口判断服务状态：
```bash
# 检测HTTP服务（适用于nginx/spring boot等）
curl -s --connect-timeout 5 http://<IP>:80/
# 检测常用端口
for port in 80 8080 3000 3306 443; do
  nc -zv -w 3 <IP> $port 2>&1 | grep -v " succeeded"
done
```

### scp 部署踩坑（2026-04-21 新增）

**问题1：`~` 路径不展开**
```bash
# 错误：scp 不展开 ~
scp -r ~/monbo-bpm/monbo-bpm-ui/dist/* root@IP:/usr/share/nginx/html/
# 正确：使用绝对路径
scp -r /Users/lirui/monbo-bpm/monbo-bpm-ui/dist/* root@IP:/usr/share/nginx/html/
```

**问题2：目标文件是 symlink，scp 无法覆盖**
```bash
# 现象：scp 成功但文件内容不变
# 原因：/usr/share/nginx/html/index.html -> ../../testpage/index.html (软链接)
# 解决：先删 symlink 再 scp
ssh root@IP "rm -f /usr/share/nginx/html/index.html"
scp /local/path/index.html root@IP:/usr/share/nginx/html/
```

**推送顺序建议**
```bash
# 1. 先删 symlink
ssh root@IP "rm -f /usr/share/nginx/html/index.html"
# 2. 再传文件
scp index.html assets/ root@IP:/usr/share/nginx/html/
```

### 已知服务器凭证（BOSS个人资产）
- **101.126.89.23** — CentOS Stream 9，root/Lirui123456，sshpass可用；已安装MySQL 8/Redis/Java 17/Nginx；GitHub Actions CI/CD已配置（RSA密钥认证）

### UI 前端部署路径
- 静态资源目录：`/opt/monbo-bpm/ui/dist/`
- 部署命令：`scp -r /Users/lirui/monbo-bpm/monbo-bpm-ui/dist/* root@101.126.89.23:/opt/monbo-bpm/ui/dist/`

### GitHub Actions CI/CD 踩坑记录（2026-04-23）

**ED25519密钥 + webfactory/ssh-agent失败**
- 现象：`Error loading key: error in libcrypto`
- 根因：webfactory/ssh-agent@v0.9.0 与 ED25519 密钥不兼容
- 解决：服务器生成 RSA 密钥对

**Java版本不匹配**
- pom.xml 配置了 `<release>21</release>`，GitHub Actions ubuntu-latest 只支持 Java 17
- 解决：统一改为 Java 17

**pnpm安装方式**
- corepack enable 不稳定
- 解决：`npm install -g pnpm`

**JAR部署原子性**
- 直接覆盖可能损坏运行中的 JAR
- 解决：先写到 tmp 目录，再 mv 原子替换

**scp大文件传输超时**
- 解决：加 `-o ConnectTimeout=60 -o ServerAliveInterval=60`

### GitHub Actions CI/CD 前后端分离部署（2026-04-23）

**问题**：job级`if`条件里用`github.event.head_commit.modified_files`在push事件下不可用（只对pull_request有效），导致workflow被跳过。

**正确方案**：拆分为两个独立workflow文件，用`on.push.paths`过滤

```yaml
# deploy-frontend.yml
on:
  push:
    branches: [main]
    paths:
      - 'monbo-bpm-ui/**'
      - '.github/workflows/deploy-frontend.yml'
  workflow_dispatch:
```

```yaml
# deploy-backend.yml
on:
  push:
    branches: [main]
    paths:
      - 'monbo-bpm-api/**'
      - '.github/workflows/deploy-backend.yml'
  workflow_dispatch:
```

**路径变更触发规则**：
- 改`monbo-bpm-ui/` → 只触发frontend workflow
- 改`monbo-bpm-api/` → 只触发backend workflow
- 改`.github/workflows/` → 两者都触发
- `workflow_dispatch`（手动）→ 两者都触发

### GitHub Actions CI/CD rsync 优化（2026-04-23）

**SCP vs rsync 选择**
- SCP：大文件可靠，但无断点续传，中断即损坏
- rsync：支持断点续传+增量同步，首次传输差不多，但修复损坏/增量部署快30-50%

**推荐 workflow 部署命令**
```yaml
# UI：rsync + checksum 确保完整
rsync -avz --delete \
  -e "ssh -o StrictHostKeyChecking=no -o ConnectTimeout=60" \
  --checksum \
  ./dist/ user@host:/path/dist-new/

# JAR：rsync + atomic mv
rsync -avz --progress \
  -e "ssh -o StrictHostKeyChecking=no -o ConnectTimeout=120" \
  ./target/*.jar user@host:/path/api-new/app.jar

# 原子替换（mv是原子操作）
ssh user@host "
  rm -f /path/api/app.jar.bak
  mv /path/api/app.jar /path/api/app.jar.bak 2>/dev/null || true
  mv /path/api-new/app.jar /path/api/app.jar
  rm -rf /path/api-new
"
```

**rsync 注意事项**
- `--checksum`：按文件内容校验，忽略 mtime 差异，确保UI完整同步
- `--progress`：显示传输进度
- 目录末尾加 `/`：传输目录内容而非目录本身

### 服务器残留文件清理

**CI/CD 中断后服务器可能残留**
```bash
rm -rf /opt/monbo-bpm/api-new
rm -f /opt/monbo-bpm/api/app.jar.bak
```

**SSH known_hosts 变更导致连接失败**
- 现象：`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`
- 解决：`ssh-keygen -R <IP>` 清除旧host key，再重连

### SSL/TLS 排查经验（2026-04-23）

**curl LibreSSL SSL_ERROR_SYSCALL 误报**
- 现象：curl 报 `LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to domain:443`
- 误导性：容易让人以为是SSL证书或nginx配置问题
- 实际：SSL可能已正常工作，需用其他工具验证
- 验证方法：
  ```bash
  # 方法1：本地curl到localhost（排除网络层问题）
  ssh root@IP "curl -vk --resolve domain:443:127.0.0.1 https://domain/"
  # 方法2：openssl s_client
  echo | openssl s_client -connect IP:443 -servername domain
  # 方法3：nginx -T 查看完整配置
  ssh root@IP "nginx -T | grep -A 5 'ssl_certificate'"
  ```
- 真实问题排查路径：
  1. `nginx -t` — 配置语法是否正确
  2. `nginx -T` — 确认server_name+ssl_certificate绑定到了正确的server block
  3. `ss -tlnp | grep 443` — nginx是否在监听443
  4. `curl -vk --resolve domain:443:127.0.0.1 https://domain/` — 本地SSL是否通
  5. 如果SSL通但返回500 — 问题在后端（检查JAR完整性、8080端口服务状态）
  6. 如果返回502/503 — 后端proxy配置问题

**JAR损坏快速诊断**
- 症状：`Error: Invalid or corrupt jarfile`
- 原因：SCP/RSYNC传输中断
- 验证：`file /path/to/app.jar` 正常应为 `Java archive data (JAR)`，损坏后可能变成 `data`
- 解决：重新触发CI/CD构建，或手动rsync重新上传

