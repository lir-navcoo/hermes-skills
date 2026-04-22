---
name: github-push-workaround
description: GitHub push失败解决 - SSH公钥不通时切换到HTTPS + GIT_TERMINAL_PROMPT=0
category: devops
tags: [git, github, ssh, push]
---

# GitHub Push失败解决

## 问题
SSH方式push时报`Permission denied (publickey)`，HTTPS方式push时报网络超时。

## 解决步骤

### 1. 先尝试SSH
```bash
git remote set-url origin git@github.com:lir-navcoo/monbo-bpm.git
GIT_SSH_COMMAND="ssh -o StrictHostKeyChecking=no -o ConnectTimeout=30" git push origin dev
```

### 2. SSH失败后切换到HTTPS + cache凭证
```bash
git remote set-url origin https://github.com/lir-navcoo/monbo-bpm.git
git config --global credential.helper "cache"
GIT_TERMINAL_PROMPT=0 git push origin dev
```

### 3. 如果仍然超时（"Failed to connect to github.com port 443"）
网络临时故障，稍等后重试即可。命令不变：
```bash
GIT_TERMINAL_PROMPT=0 git push origin dev
```

## 关键参数

| 参数 | 作用 |
|------|------|
| `GIT_SSH_COMMAND="ssh -o ..."` | SSH方式，设置超时和跳过host检查 |
| `GIT_TERMINAL_PROMPT=0` | HTTPS方式，禁用git的交互式密码提示 |
| `credential.helper "cache"` | HTTPS方式，将凭证缓存memory（仅当前session）|

## 凭证存储位置（可选，长期方案）
```bash
git config --global credential.helper store
# 下次push会提示输入用户名/token，之后存储在 ~/.git-credentials
```

## 最可靠方案：expect + gh auth token

当SSH和普通HTTPS都无法推送时（git报"terminal prompts disabled"或"Device not configured"），使用expect自动输入token：

```bash
# 确保remote是HTTPS
git remote set-url origin https://github.com/lir-navcoo/monbo-bpm-ui.git

# expect自动填入用户名和token
expect -c '
spawn git push -u origin main:test
expect "Username for *github.com*"
send "lir-navcoo\r"
expect "Password for *github.com*"
send "'"$(gh auth token)"'\r"
expect eof
'
```

原理：HTTPS push需要用户名+GitHub Personal Access Token认证，用`gh auth token`获取token，通过expect自动填入密码。

## 凭证存储位置（可选，长期方案）
```bash
git config --global credential.helper store
# 下次push会提示输入用户名/token，之后存储在 ~/.git-credentials
```

## 当前monbo-bpm仓库push状态
- origin已设置为HTTPS: `https://github.com/lir-navcoo/monbo-bpm-ui.git`
- 每次push需要使用expect + gh auth token方式推送
- SSH方式完全被阻断（Connection closed by 198.18.0.12）
- 优先使用expect + gh auth token，这是最可靠的方式

### 步骤

```python
import json, base64, subprocess

GH_REPO = "lir-navcoo/monbo-bpm"
# 获取auth token
token = subprocess.run("gh auth token", shell=True, capture_output=True, text=True).stdout.strip()

# 读取文件
content = open("file.tsx").read()
encoded = base64.b64encode(content.encode()).decode()

# 1. 创建blob
result = subprocess.run(
    f"curl -s -X POST -H 'Authorization: Bearer {token}' -H 'Content-Type: application/json' "
    f"-d @- https://api.github.com/repos/{GH_REPO}/git/blobs",
    shell=True, capture_output=True, text=True,
    input=json.dumps({"content": encoded, "encoding": "base64"})
)
blob_sha = json.loads(result.stdout)["sha"]

# 2. 获取parent commit的tree_sha
parent_result = subprocess.run(
    f"curl -s -H 'Authorization: Bearer {token}' "
    f"https://api.github.com/repos/{GH_REPO}/git/commits/{PARENT_SHA}",
    shell=True, capture_output=True, text=True
)
tree_sha = json.loads(parent_result.stdout)["tree"]["sha"]

# 3. 创建新tree
tree_result = subprocess.run(
    f"curl -s -X POST -H 'Authorization: Bearer {token}' -H 'Content-Type: application/json' "
    f"-d @- https://api.github.com/repos/{GH_REPO}/git/trees",
    shell=True, capture_output=True, text=True,
    input=json.dumps({"base_tree": tree_sha, "tree": [{"path": "path/to/file.tsx", "mode": "100644", "type": "blob", "sha": blob_sha}]})
)
new_tree_sha = json.loads(tree_result.stdout)["sha"]

# 4. 创建commit
commit_result = subprocess.run(
    f"curl -s -X POST -H 'Authorization: Bearer {token}' -H 'Content-Type: application/json' "
    f"-d @- https://api.github.com/repos/{GH_REPO}/git/commits",
    shell=True, capture_output=True, text=True,
    input=json.dumps({"message": "commit message", "tree": new_tree_sha, "parents": [PARENT_SHA]})
)
new_commit_sha = json.loads(commit_result.stdout)["sha"]

# 5. 更新分支ref
subprocess.run(
    f"curl -s -X POST -H 'Authorization: Bearer {token}' -H 'Content-Type: application/json' "
    f"-d @- https://api.github.com/repos/{GH_REPO}/git/refs/heads/dev",
    shell=True, capture_output=True, text=True,
    input=json.dumps({"sha": new_commit_sha})
)
```

### 关键点
- `gh auth token` 获取的token可直接用于curl的Authorization header
- blob必须用base64编码，encoding字段设为"base64"
- 适用于本地无法git push但需要紧急推送的场景

## 当前monbo-bpm仓库push状态
- origin已设置为HTTPS: `https://github.com/lir-navcoo/monbo-bpm.git`
- 每次push需要使用`GIT_TERMINAL_PROMPT=0`避免交互式提示
- 网络不通时代码暂存本地，等待网络恢复后push
