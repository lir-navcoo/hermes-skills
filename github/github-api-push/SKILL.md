---
name: github-api-push
description: GitHub API 文件推送工作流 — 包含创建空仓库初始化、批量多文件推送、GitHub Pages 配置、用空 commit 触发 Actions rebuild。本地 build 验证是强制步骤。
---

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

## ⚠️ 关键坑点1：更新已有文件必须提供 SHA
PUT 新文件不需要 SHA，但**更新已存在的文件必须先查 SHA 再推送**，否则报 422：
```bash
# 第一步：查询 SHA（文件存在则返回值，不存在则 404）
SHA=$(gh api "repos/{owner}/{repo}/contents/{path}?ref=main" --jq '.sha' 2>/dev/null)

# 第二步：根据是否存在决定是否加 sha 参数
if [ "$SHA" = "Not Found" ] || [ -z "$SHA" ]; then
    # 新文件或罕见情况：不传 sha
    gh api --method PUT "repos/{owner}/{repo}/contents/{path}" \
      -f message="add: {path}" -f content="$CONTENT"
else
    # 已有文件：必须传 sha
    gh api --method PUT "repos/{owner}/{repo}/contents/{path}" \
      -f message="update: {path}" -f content="$CONTENT" -f sha="$SHA"
fi
```

## ⚠️ 关键坑点2：Tree API 会双重编码 content（已造成303个文件全部损坏）
**严重问题**：批量多文件推送时，tree API 的 `content` 字段会被 API 内部自动再做一层 base64 编码。**实测结果：所有303个文件全部损坏，MD5与原始内容完全不同**。绝对不能在 tree 的 `content` 字段传 base64 字符串。

**正确方案：Tree API 必须分两步走（blob-first）**

第一步 — 创建 blob 获取 SHA：
```python
import subprocess, base64

def create_blob(repo, file_path):
    with open(file_path, 'rb') as f:
        encoded = base64.b64encode(f.read()).decode('ascii')
    result = subprocess.run(
        ['gh', 'api', f'repos/{repo}/git/blobs', '--method', 'POST',
         '-f', f'content={encoded}', '-f', 'encoding=base64'],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)['sha']
```

第二步 — 用 blob SHA 构建 tree（不是 content）：
```python
tree = [{'path': rel_path, 'mode': '100644', 'type': 'blob', 'sha': blob_sha}]
```

**注意**：`type` 必须是 `'blob'`（不是 `'file'`），且用 `sha` 引用而非 `content`。

**简单方案（推荐）**：大批量同步时，改用**逐文件单次 PUT** 推送，详见下方 Python 脚本。
```python
import subprocess, os, base64, json

repo = "owner/repo"
base_path = "/path/to/local/skills"
skip_dirs = {'.git', 'node_modules', '.DS_Store'}

# 遍历所有本地文件
for root, dirs, files in os.walk(base_path):
    dirs[:] = [d for d in dirs if d not in skip_dirs]
    for fname in files:
        fpath = os.path.join(root, fname)
        rel = os.path.relpath(fpath, base_path)
        with open(fpath, 'r', encoding='utf-8') as f:
            content = f.read()
        encoded = base64.b64encode(content.encode('utf-8')).decode('ascii')
        
        # 查询现有 SHA（处理新文件 vs 更新）
        check = subprocess.run(
            ['gh', 'api', f'repos/{repo}/contents/{rel}?ref=main', '--jq', '.sha'],
            capture_output=True, text=True
        )
        sha = check.stdout.strip()
        msg = f"sync: {rel}"
        
        if check.returncode != 0 or not sha or 'sha' not in check.stdout:
            # 新文件
            subprocess.run(['gh', 'api', '--method', 'PUT', f'repos/{repo}/contents/{rel}',
                           '-f', f'message={msg}', '-f', f'content={encoded}'])
        else:
            # 已有文件，需 SHA
            subprocess.run(['gh', 'api', '--method', 'PUT', f'repos/{repo}/contents/{rel}',
                           '-f', f'message={msg}', '-f', f'content={encoded}', '-f', f'sha={sha}'])
```

## ⚠️ 关键坑点3：GitHub 文件 size 不等于本地 size，且 size 不能用于同步验证
GitHub 对文件内容会额外编码，**同一个文件在 GitHub 显示的 size 会比本地大**（UTF-8 文本大约大 10-20%，非ASCII内容差异更大）。**绝对不能通过 size 对比来判断同步是否成功**，否则会漏掉所有损坏的文件（如本次303个文件全部损坏但size差异固定在33%左右）。

**正确的验证方式：用 md5 抽查关键文件**：
```bash
# 本地 md5
md5 ~/.hermes/skills/apple/apple-reminders/SKILL.md

# GitHub md5（下载后验证）
gh api repos/{owner}/{repo}/contents/apple/apple-reminders/SKILL.md \
  --jq '.content' | base64 -d | md5

# 用 diff 对比（最可靠）
gh api repos/{owner}/{repo}/contents/apple/apple-reminders/SKILL.md \
  --jq '.content' | base64 -d > /tmp/gh_file.md
diff ~/.hermes/skills/apple/apple-reminders/SKILL.md /tmp/gh_file.md
```

**同步完整性验证脚本**：
```python
import subprocess, os, hashlib, base64, json

def md5_local(path):
    with open(path, 'rb') as f:
        return hashlib.md5(f.read()).hexdigest()

def md5_remote(repo, path):
    result = subprocess.run(
        ['gh', 'api', f'repos/{repo}/contents/{path}', '--jq', '.content'],
        capture_output=True, text=True
    )
    return hashlib.md5(base64.b64decode(result.stdout.strip())).hexdigest()
```

## ⚠️ 旧版扁平结构 vs 新版目录结构
早期同步曾将 skills 扁平化推送（如 `astock-analysis.md`），后改为目录结构（如 `finance/astock-analysis/SKILL.md`）。同一仓库中可能存在两种结构并存、文件路径冲突、旧文件多余等情况。排查差异时用 tree recursive API 获取完整文件列表：
```bash
gh api repos/{owner}/{repo}/git/trees/main?recursive=1
```

## ⚠️ 关键坑点4：docs/ 文件夹必须先查 SHA
GitHub Pages 从 `docs/` 目录托管文件时，即使是新文件也必须先查 SHA 再推送（否则报 422）：

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

## 批量推送多文件（推荐逐文件PUT，tree API需用blob-first）
**重要**：由于 tree API 的 `content` 字段存在双重编码问题，**强烈建议用逐文件 PUT 方式**推送。tree API 仅在需要原子性提交大量文件时才用，且必须用 blob-first 写法。

### 推荐：逐文件PUT（安全可靠）
```python
import subprocess, os, base64, json

repo = "owner/repo"
base_path = "/path/to/local/skills"
skip_dirs = {'.git', 'node_modules', '.DS_Store'}

for root, dirs, files in os.walk(base_path):
    dirs[:] = [d for d in dirs if d not in skip_dirs]
    for fname in files:
        if not fname.endswith('.md'):
            continue
        fpath = os.path.join(root, fname)
        rel = os.path.relpath(fpath, base_path)

        # 读取并编码
        with open(fpath, 'rb') as f:
            encoded = base64.b64encode(f.read()).decode('ascii')

        # 查询现有 SHA
        check = subprocess.run(
            ['gh', 'api', f'repos/{repo}/contents/{rel}?ref=main', '--jq', '.sha'],
            capture_output=True, text=True
        )
        sha = check.stdout.strip()
        msg = f"sync: {rel}"

        if check.returncode != 0 or not sha or 'sha' not in check.stdout:
            subprocess.run(
                ['gh', 'api', '--method', 'PUT', f'repos/{repo}/contents/{rel}',
                 '-f', f'message={msg}', '-f', f'content={encoded}']
            )
        else:
            subprocess.run(
                ['gh', 'api', '--method', 'PUT', f'repos/{repo}/contents/{rel}',
                 '-f', f'message={msg}', '-f', f'content={encoded}', '-f', f'sha={sha}']
            )
        print(f"Pushed: {rel}")
```

### 仅在需要时用：Tree API blob-first（正确写法）
```python
import json, subprocess, os, base64

repo = "owner/repo"
base_path = "/path/to/project"
skip_dirs = {'node_modules', '.git', 'dist'}

def create_blob(repo, path):
    with open(path, 'rb') as f:
        encoded = base64.b64encode(f.read()).decode('ascii')
    result = subprocess.run(
        ['gh', 'api', f'repos/{repo}/git/blobs', '--method', 'POST',
         '-f', f'content={encoded}', '-f', 'encoding=base64'],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)['sha']

# 获取当前 HEAD
head_sha = subprocess.run(
    ['gh', 'api', f'repos/{repo}/git/ref/heads/main', '--jq', '.object.sha'],
    capture_output=True, text=True
).stdout.strip()

# 先创建所有 blobs
blob_map = {}
for root, dirs, files in os.walk(base_path):
    dirs[:] = [d for d in dirs if d not in skip_dirs]
    for fname in files:
        fpath = os.path.join(root, fname)
        rel = os.path.relpath(fpath, base_path)
        blob_map[rel] = create_blob(repo, fpath)
        print(f"Blob created: {rel}")

# 用 blob SHA 构建 tree（type='blob' 且用 sha 引用）
tree = [
    {'path': rel, 'mode': '100644', 'type': 'blob', 'sha': sha}
    for rel, sha in blob_map.items()
]

# 创建新 tree
tree_payload = json.dumps({"tree": tree})
with open('/tmp/tree_payload.json', 'w') as f:
    f.write(tree_payload)

r = subprocess.run(
    ['gh', 'api', f'repos/{repo}/git/trees', '--method', 'POST', '--input', '/tmp/tree_payload.json'],
    capture_output=True, text=True
)
new_tree_sha = json.loads(r.stdout)['sha']

# 创建 commit
commit_payload = json.dumps({"message": "sync: batch update", "tree": new_tree_sha, "parents": [head_sha]})
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
print(f"Done: {len(blob_map)} files synced")
```

**关键**：`tree[].type` 必须是 `'blob'`，且用 `sha` 引用而非 `content`。

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

## 排查远程仓库：检查文件是否存在 / 差异对比

### 查看目录内容（列出文件/子目录）
```bash
# 目录 path 不带尾部斜杠
gh api repos/{owner}/{repo}/contents/{path} --jq '.[].name,.[].type'
# 例如：列出根目录
gh api repos/lir-navcoo/hermes-skills/contents/ --jq '.[].name'
# 例如：列出 finance 子目录
gh api repos/lir-navcoo/hermes-skills/contents/finance --jq '.[].name'
# 例如：查看某个 skill 目录里的文件
gh api repos/lir-navcoo/hermes-skills/contents/finance/astock-analysis --jq '.[].name'
```

### 检查单个文件是否存在 + 读取内容
```bash
# 检查文件 SHA（存在则返回 sha，不存在则 404）
gh api repos/{owner}/{repo}/contents/{path} --jq '.sha'

# 直接读取文件内容（自动返回 base64，需解码）
gh api repos/{owner}/{repo}/contents/{path} --jq '.content' | base64 -d
# 示例：读取远程 himalaya.md
gh api repos/lir-navcoo/hermes-skills/contents/himalaya.md --jq '.content' | base64 -d | head -20
```

### 排查本地 vs 远程差异
```bash
# 本地有但 GitHub 没有的文件（本地 vs GitHub 根目录）
comm -23 <(ls ~/.hermes/skills/ | sort) \
        <(gh api repos/lir-navcoo/hermes-skills/contents/ --jq '.[].name' | sort)

# 对比特定目录
diff <(ls ~/.hermes/skills/finance/ | sort) \
     <(gh api repos/lir-navcoo/hermes-skills/contents/finance --jq '.[].name' | sort)
```

### 关键坑点
- **422 + "sha wasn't supplied"** = 文件在 GitHub 已存在，更新时必须传 sha 参数
- **404** = 文件或目录在远程不存在
- **GitHub 显示的 size ≠ 本地原始 size**，不能用于验证同步正确性
- 查询目录时 path 不能有尾部斜杠
- 批量推送优先用**逐文件单次 PUT**，避免 tree API 双重编码问题

## Git Clone（仅 build 验证时用）
```bash
# 单文件下载
gh api "repos/{owner}/{repo}/contents/{path}?ref=main" --jq '.content' | base64 -d > /tmp/file
# 完整 clone
git clone https://TOKEN@github.com/{owner}/{repo}.git
```
注意：clone 时 remote URL 需要内嵌 TOKEN，格式：`https://TOKEN@github.com/user/repo.git`
