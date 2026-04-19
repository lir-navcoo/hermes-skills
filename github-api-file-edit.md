
# GitHub API File Edit Workflow

修改 GitHub 上仓库文件时，如无本地克隆可用，通过 GitHub REST API 直接操作。

## 适用场景
- 本地无项目克隆
- 只想修改几个文件而不必 clone 整个仓库
- 临时快速推送改动

## 标准流程

### 1. 获取文件 SHA 和当前内容
```bash
gh api "repos/{owner}/{repo}/contents/{path}?ref={branch}" --jq '.sha, .content'
```
返回 `{sha}` 和 `{content}`（base64编码）

### 2. 下载文件内容（需 decode）
```bash
gh api "repos/{owner}/{repo}/contents/{path}?ref={branch}" --jq '.content' | base64 -d > local_file
```

### 3. 修改文件后推送
```bash
SHA=$(gh api "repos/{owner}/{repo}/contents/{path}?ref={branch}" --jq '.sha')
CONTENT=$(base64 -i local_file)
gh api --method PUT "repos/{owner}/{repo}/contents/{path}" \
  -f message="commit message" \
  -f content="$CONTENT" \
  -f sha="$SHA"
```

### 4. 读取目录结构（树形）
```bash
gh api "repos/{owner}/{repo}/git/trees/{branch}?recursive=1" --jq '.tree[].path'
```

### 5. GraphQL 查询（如 pinned items）
```bash
gh api graphql -f query='{ user(login: "username") { pinnedItems(first: 6, types: REPOSITORY) { nodes { ... on Repository { name description url primaryLanguage { name } } } } } }' --jq '.data.user.pinnedItems.nodes[]'
```

## 坑点

- `gh api ... --jq '.content'` 拿到的 content 是 base64 编码的，不是明文
- 用 `patch` 工具做字符串替换时，必须精确匹配（包括空格、缩进），复杂文件容易失败
- 大面积改动时：write_file 写到 `/tmp/`，再 base64 推上去，比 patch 更可靠
- SHA 必须用最新值，旧的会报 409 Conflict
- 路径不存在时（如分支/目录错误）会返回 404

## 权限
需 GitHub PAT（git credential 方式：remote URL 嵌入 TOKEN，或 `gh auth login`）
