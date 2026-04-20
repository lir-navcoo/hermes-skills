---
name: obsidian
description: Read, search, and create notes in the Obsidian vault.
---

# Obsidian Vault

**Location:** Set via `OBSIDIAN_VAULT_PATH` environment variable (e.g. in `~/.hermes/.env`).

If unset, defaults to `~/Documents/Obsidian Vault`.

Note: Vault paths may contain spaces - always quote them.

## Read a note

```bash
VAULT="${OBSIDIAN_VAULT_PATH:-$HOME/Documents/Obsidian Vault}"
cat "$VAULT/Note Name.md"
```

## List notes

```bash
VAULT="${OBSIDIAN_VAULT_PATH:-$HOME/Documents/Obsidian Vault}"

# All notes
find "$VAULT" -name "*.md" -type f

# In a specific folder
ls "$VAULT/Subfolder/"
```

## Search

```bash
VAULT="${OBSIDIAN_VAULT_PATH:-$HOME/Documents/Obsidian Vault}"

# By filename
find "$VAULT" -name "*.md" -iname "*keyword*"

# By content
grep -rli "keyword" "$VAULT" --include="*.md"
```

## Create a note

```bash
VAULT="${OBSIDIAN_VAULT_PATH:-$HOME/Documents/Obsidian Vault}"
cat > "$VAULT/New Note.md" << 'ENDNOTE'
# Title

Content here.
ENDNOTE
```

## Append to a note

```bash
VAULT="${OBSIDIAN_VAULT_PATH:-$HOME/Documents/Obsidian Vault}"
echo "
New content here." >> "$VAULT/Existing Note.md"
```

## Obsidian 标签限制（重要）

Obsidian 的 tags 不支持：
- 纯数字（如 `002195`）→ 会报 "unknown data incompatible" 错误
- 某些特殊字符

**解决方案**：数字前加前缀字符

```yaml
# 错误示例
tags:
  - 持仓        # ❌ 纯中文在某些版本可能有问题
  - 002195     # ❌ 纯数字不支持

# 正确示例（BOSS验证可行）
tags:
  - 持仓
  - 岩山科技
  - 股002195   # ✅ 中文字符 + 数字组合可行
```

## 文件命名规范

- 索引文件命名：文件夹名.md（如 `标的追踪.md` 而不是 `标的追踪索引.md`）
- 持仓文件命名：`代码-名称.md`（如 `002097-山河智能.md`）
- 推荐追踪文件命名：`YYYY-MM-DD-代码-名称.md`

## Wikilinks

Obsidian links notes with `[[Note Name]]` syntax. When creating notes, use these to link related content.
