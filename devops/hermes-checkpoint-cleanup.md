---
name: hermes-checkpoint-cleanup
description: 清理Hermes Agent残留的checkpoint、worktree和进程，解决agent-webui等已删除skill仍在运行的问题
version: 1.0.0
author: 多宝
license: MIT
---

# Hermes Checkpoint & 残留进程清理

## 症状
- skill已删除但相关进程仍在运行（如agent-webui）
- 端口（如18765）仍被占用
- ~/agent-webui目录反复自动出现

## 排查步骤

### 1. 找进程
```bash
lsof -i :PORT  # 如 lsof -i :18765
ps aux | grep -i SKILL_NAME | grep -v grep
```

### 2. 杀进程
```bash
lsof -ti :PORT | xargs kill -9 2>/dev/null
```

### 3. 删残留目录
```bash
rm -rf ~/AGENT_WEBUI_DIR  # 如 ~/agent-webui
```

### 4. 查LaunchAgents
```bash
ls ~/Library/LaunchAgents/
ls /Library/LaunchAgents/
```
如果有残留plist，unload并删除：
```bash
launchctl unload ~/Library/LaunchAgents/XXX.plist
rm ~/Library/LaunchAgents/XXX.plist
```

### 5. 查Hermes Checkpoints（关键！）
```bash
grep -l "AGENT_WEBUI" ~/.hermes/checkpoints/*/config
```
checkpoints里存了worktree配置，必须删掉：
```bash
rm -rf ~/.hermes/checkpoints/CHECKPOINT_ID
```

### 6. 验证
```bash
lsof -i :PORT  # 无输出=干净
ps aux | grep -i SKILL_NAME | grep -v grep  # 无输出=干净
```

## 根因
Hermes在worktree模式或checkpoint中记录了skill的运行路径（如`~/agent-webui`），即使skill本身删了，checkpoint仍存在且会触发Hermes尝试重新运行。skill删除不等于进程停止。