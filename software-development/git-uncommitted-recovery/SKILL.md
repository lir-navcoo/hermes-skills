---
name: git-uncommitted-recovery
description: Recover uncommitted changes lost to git reset --hard, git clean, or accidental checkout
---

# Git Uncommitted Changes Recovery

## Problem
`git reset --hard origin/main` (or any branch) destroys ALL uncommitted local changes — new files, modified files, unstaged changes. No warning is given before destruction.

## Recovery Methods (in order of likelihood)

### Method 1: Git Reflog (BEST)
```bash
git reflog --all -20
# Find the commit BEFORE the reset (e.g. HEAD@{1})
git show HEAD@{1} --stat   # inspect what was lost
# If the files existed in a previous commit on any branch:
git ls-tree -r HEAD@{1} --name-only | grep -i "filename"
# Restore:
git checkout HEAD@{1} -- path/to/file
```

### Method 2: Git Stash (if stashed before reset)
```bash
git stash list
git stash show -p stash@{0}
git stash apply stash@{0}
```

### Method 3: File System Recovery
- macOS: Time Machine backup
- IDE: Local History (WebStorm, VSCode)
- Spotlight: search by filename

## Prevention Rules (MUST FOLLOW)

1. **Before ANY `git reset --hard`, `git clean -fd`, or `git checkout --`**: ALWAYS `git stash` or `git commit` first
2. **Work-in-progress commits**: commit with `WIP:` prefix, can be amended/squashed later
3. **Critical refactorings**: push to remote branch immediately, not just local

## Trigger Condition
When user says "re-pull branch", "reset to clean state", "checkout clean version" — ALWAYS ask "should I stash first?" before executing destructive commands. Do NOT assume the user wants to destroy uncommitted work.
