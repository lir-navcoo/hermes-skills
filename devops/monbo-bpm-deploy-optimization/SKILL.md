---
name: monbo-bpm-deploy-optimization
description: monbo-bpm 后端部署优化记录 - CI rsync 直传 JAR vs GitHub CDN 下载
---

# monbo-bpm 部署优化记录

## 后端部署方案（2026-04-24 优化）

**问题**：GitHub CDN 下载 JAR（~55MB）需要 10+ 分钟，CI 总时长 13分54秒

**原因**：服务器（阿里云 101.126.89.23）在国内访问 GitHub 国际出口带宽慢

**优化方案**：CI runner 构建 → rsync 直传 JAR 到服务器

**效果**：4分23秒，比原来快 3 倍

### 关键 workflow 配置

```yaml
- name: Build & Deploy to Server
  run: |
    cd monbo-bpm
    mvn package -DskipTests -q
    rsync -az --progress \
      -e "ssh -o StrictHostKeyChecking=no" \
      target/monbo-bpm-0.0.1-SNAPSHOT.jar \
      root@101.126.89.23:/opt/monbo-bpm/monbo-bpm.jar
```

**坑**：
- 服务器需要能 SSH 到 GitHub（实际不行，国内墙）
- 用 CI runner 作为 rsync 源，点对点传输无国际出口瓶颈
- 需要 `known_hosts` 验证：先 `ssh-keyscan github.com` 添加 known_hosts

## 前端部署（保持不变）

```bash
sshpass -p 'Lirui123456' scp -o StrictHostKeyChecking=no \
  -r dist/* root@101.126.89.23:/opt/monbo-bpm/ui/
```

## 服务器信息

- IP: 101.126.89.23
- 前端: /opt/monbo-bpm/ui/
- 后端 JAR: /opt/monbo-bpm/monbo-bpm.jar
- OS: CentOS Stream 9
