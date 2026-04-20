---
name: memory-management
description: Memory capacity, cleanup strategy, and what to save/drop
---

# Memory Management

## Capacity
- Hard limit: **2,200 characters**
- Memory is injected into every turn — keep it compact

## What to Save (high value, durable)
- User preferences, corrections, habits
- Environment facts, project conventions, tool quirks
- Learned approaches that prevent future steering

## What NOT to Save
- Task progress, session outcomes, completed-work logs
- Raw data dumps, temporary TODO state
- Verbose session summaries (these eat space fast)

## When Memory Gets Full
When `add` fails with "exceed limit", do NOT use `replace` — use `remove` on old entries first.
Priority order for removal:
1. "Conversation Summary: assistant: ..." entries (auto-populated, verbose)
2. "Conversation Summary: ..." with detailed multi-line summaries
3. Old status/echo entries that are just repeat output
4. Keep: user profile, tool-learned approaches, project conventions

## Cleanup Strategy
- Memory space: ~100 chars per conversation summary entry
- Remove entries one at a time with `memory(action='remove', old_text='...')`
- After cleanup, verify with `memory(action='list')` or just try adding

## External File Offloading（推荐）

当条目内容超过200字符时，改为存文件+记忆存索引：

1. 大内容写入 `~/.hermes/knowledge/<topic>.md`
2. 记忆中只存一行引用：`主题 → ~/.hermes/knowledge/<topic>.md`
3. 读取时用 `read_file("~/.hermes/knowledge/<topic>.md")`

**适用场景：** 技术栈规范、长篇自学记录、项目细节

**操作示例：**
- 写入文件：`write_file("~/.hermes/knowledge/tech-stack.md", content)`
- 更新记忆：`memory(action='replace', old_text='完整技术栈文本', new_string='技术栈规范 → ~/.hermes/knowledge/tech-stack.md')`

## 记忆操作踩坑

**replace失败"No entry matched"：** 字符串必须完全匹配原文，包括空格、换行符、标点。宁可删了重建，不要执着于replace。

**操作顺序：**
1. `remove` 旧条目（即使replace失败也可以remove掉）
2. `add` 新条目
不要在同一次调用里同时做两件事。

## Practical Thresholds
- Under 50% (1,100 chars): comfortable
- 50-80%: watch for accumulation
- Above 80%: proactively clean before adding new content
- At limit: remove old session summaries first, then retry
