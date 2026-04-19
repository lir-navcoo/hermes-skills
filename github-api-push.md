
# GitHub API 文件推送工作流

## 背景
`gh auth git-credential` 在某些环境下不工作，导致 `git push` 报 "could not read Password: Device not configured"。解决方案：绕开 git，直接用 GitHub REST API 推送文件。

## ⚠️ 关键原则：必须手动 Base64 编码
GitHub API 的 `content` 字段期望 **Base64 编码后的字符串**，不是原始文本。
如果不 base64 直接传，API 报 `content is not valid Base64` (HTTP 422)。

正确做法（Python）：
```python
import base64
encoded = base64.b64encode(raw_content.encode('utf-8')).decode('ascii')
subprocess.run(['gh', 'api', '--method', 'PUT', f'repos/{repo}/contents/{path}',
                '-f', f'message={msg}', '-f', f'content={encoded}'])
```

Shell 中可接受（但不如 Python 可靠）：
```bash
CONTENT=$(base64 -i /path/to/file | tr -d '\n')
gh api --method PUT "repos/.../contents/path" \
  -f message="..." -f content="$CONTENT"
```

## 推送流程（已有文件的更新）

### 第一步：获取当前 SHA
```bash
SHA=$(gh api "repos/{owner}/{repo}/contents/{path}?ref={branch}" --jq '.sha')
```

### 第二步：Base64 编码
```bash
CONTENT=$(base64 -i /path/to/file | tr -d '\n')
```

### 第三步：PUT 推送
```bash
gh api --method PUT "repos/{owner}/{repo}/contents/{path}" \
  -f message="commit message" \
  -f content="$CONTENT" \
  -f sha="$SHA"
```

## ⚠️ GitHub Pages 路径限制
`/dist` 不是有效路径，只能用 `/`（根目录）或 `/docs`。如果项目 build 输出在 `dist/`，需要用 `actions/upload-pages-artifact` + `actions/deploy-pages`（而非 `actions/upload-artifact`），且 Pages source 设置为 `/`。

## 完整模板
```bash
SHA=$(gh api "repos/USER/REPO/contents/src/App.tsx?ref=main" --jq '.sha')
CONTENT=$(base64 -i /tmp/App.tsx | tr -d '\n')
gh api --method PUT "repos/USER/REPO/contents/src/App.tsx" \
  -f message="your commit message" \
  -f content="$CONTENT" \
  -f sha="$SHA"
```

## ⚠️ 关键坑点：`docs/` 文件夹必须先查 SHA
GitHub Pages 从 `docs/` 目录托管文件时，即使是新文件也必须先查 SHA 再推送（否则报 422）：
```bash
# 先尝试获取 SHA（文件存在则返回，不存在则报错）
SHA=$(gh api "repos/USER/REPO/contents/docs/avatar.jpg?ref=main" --jq '.sha' 2>/dev/null)
# 如果上面报错（文件不存在），换用以下方式：
SHA=$(gh api "repos/USER/REPO/contents/docs/avatar.jpg?ref=main" --jq '.sha' || echo "NEW_FILE")
```

## ⚠️ GitHub Actions Pages Build 日志不对外暴露
若 build 失败需要排查，必须 clone 到本地 build 验证，GitHub 不提供 Actions 日志公开 API。

## 创建空仓库

```bash
# 正确 endpoint: /user/repos（不是 /repos）
gh api /user/repos -X POST \
  -f name="repo-name" \
  -f description="..." \
  -f visibility=public \
  -f auto_init=false
```

## 批量推送多文件（用于已初始化的仓库）
适用于需要一次性推送大量文件（如 npm install 后的 node_modules 不算，但源文件可以）的场景：
```python
import json, subprocess, os

repo = "owner/repo"
base_path = "/path/to/project"
skip_dirs = {'node_modules', '.git', 'dist'}

# 获取当前 HEAD
head_sha = subprocess.run(
    ['gh', 'api', f'repos/{repo}/git/ref/heads/main', '--jq', '.object.sha'],
    capture_output=True, text=True
).stdout.strip()

tree_sha = subprocess.run(
    ['gh', 'api', f'repos/{repo}/git/commits/{head_sha}', '--jq', '.tree.sha'],
    capture_output=True, text=True
).stdout.strip()

# 构建 tree（content 传 base64 编码后的字符串）
import base64
tree = []
for root, dirs, files in os.walk(base_path):
    dirs[:] = [d for d in dirs if d not in skip_dirs]
    for fname in files:
        fpath = os.path.join(root, fname)
        rel = os.path.relpath(fpath, base_path)
        with open(fpath, 'r', encoding='utf-8') as f:
            content = f.read()
        encoded = base64.b64encode(content.encode('utf-8')).decode('ascii')
        tree.append({'path': rel, 'mode': '100644', 'type': 'blob', 'content': encoded})

# 创建新 tree
payload = json.dumps({"base_tree": tree_sha, "tree": tree})
with open('/tmp/tree_payload.json', 'w') as f:
    f.write(payload)

r = subprocess.run(
    ['gh', 'api', f'repos/{repo}/git/trees', '--method', 'POST', '--input', '/tmp/tree_payload.json'],
    capture_output=True, text=True
)
new_tree_sha = json.loads(r.stdout)['sha']

# 创建 commit
commit_payload = json.dumps({"message": "commit msg", "tree": new_tree_sha, "parents": [head_sha]})
with open('/tmp/commit_payload.json', 'w') as f:
    f.write(commit_payload)

r2 = subprocess.run(
    ['gh', 'api', f'repos/{repo}/git/commits', '--method', 'POST', '--input', '/tmp/commit_payload.json'],
    capture_output=True, text=True
)
commit_sha = json.loads(r2.stdout)['sha']

# 更新 ref
ref_payload = json.dumps({"sha": commit_sha})
with open('/tmp/ref_payload.json', 'w') as f:
    f.write(ref_payload)
subprocess.run(
    ['gh', 'api', f'repos/{repo}/git/refs/heads/main', '--method', 'PATCH', '--input', '/tmp/ref_payload.json'],
    capture_output=True, text=True
)
```

### 触发空 commit 重建 Actions
如果需要让 GitHub Actions 重新运行但不想改任何文件：
```python
r = subprocess.run(['gh', 'api', f'repos/{repo}/git/ref/heads/main', '--jq', '.object.sha'], capture_output=True, text=True)
head_sha = r.stdout.strip()
r2 = subprocess.run(['gh', 'api', f'repos/{repo}/git/commits/{head_sha}', '--jq', '.tree.sha'], capture_output=True, text=True)
base_tree_sha = r2.stdout.strip()
commit_payload = json.dumps({"message": "ci: trigger rebuild", "tree": base_tree_sha, "parents": [head_sha]})
with open('/tmp/c.json', 'w') as f: f.write(commit_payload)
r3 = subprocess.run(['gh', 'api', f'repos/{repo}/git/commits', '--method', 'POST', '--input', '/tmp/c.json'], capture_output=True, text=True)
if r3.returncode == 0:
    commit_sha = json.loads(r3.stdout)['sha']
    ref_payload = json.dumps({"sha": commit_sha})
    with open('/tmp/r.json', 'w') as f: f.write(ref_payload)
    subprocess.run(['gh', 'api', f'repos/{repo}/git/refs/heads/main', '--method', 'PATCH', '--input', '/tmp/r.json'], capture_output=True, text=True)
```

## GitHub Pages 配置（当 GitHub Pages 未启用时）
```bash
gh api repos/{owner}/{repo}/pages --method POST \
  -f source[branch]=main \
  -f build_type=workflow
```
注意：`source[path]` 只能是 `/` 或 `/docs`，不能是 `/dist`。

## 本地 Build 验证（强制）
每次改动必须先在本地跑 build，确认 TypeScript 编译通过再推送：
```bash
cd /path/to/repo
npm install 2>&1 | tail -3
npm run build 2>&1 | tail -8
```
Build 失败 → 修复后再推送，不推送有问题的代码到 main。

## Git Clone（仅 build 验证时用）
```bash
# 单文件下载
gh api "repos/{owner}/{repo}/contents/{path}?ref=main" --jq '.content' | base64 -d > /tmp/file
# 完整 clone
git clone https://TOKEN@github.com/{owner}/{repo}.git
```
注意：clone 时 remote URL 需要内嵌 TOKEN，格式：`https://TOKEN@github.com/user/repo.git`
