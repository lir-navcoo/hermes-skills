---
name: context-optimization
description: Optimize Hermes context usage — expand RTK coverage, prune sessions, manage memory, set up auto-prune cron. Use when token usage is high or RTK coverage is low.
category: software-development
---

# Context Optimization Workflow

Use this skill when Adit wants to reduce token usage, optimize context, or check RTK/agent efficiency.

## When to Use
- User asks about token savings, RTK coverage, or context management
- Periodic maintenance (monthly recommended)
- After noticing high session count or bloated database

## Steps

### 1. Check Current State
```bash
rtk gain                          # RTK token savings
hermes insights                   # Full usage analytics (last 30 days)
hermes sessions stats             # Session store size
```

### 2. Prune Old Sessions
```bash
# Cron sessions older than 3 days (auto-generated, safe to delete)
hermes sessions prune --older-than 3 --source cron --yes

# Telegram sessions older than 14 days (if needed)
hermes sessions prune --older-than 14 --source telegram --yes
```

### 3. Verify RTK Enforcement
Check that CLAUDE.md contains:
- Golden Rule: always prefix with `rtk`
- MANDATORY section for execute_code terminal() calls
- Exception list (interactive, no-output, full-output)

If missing, patch CLAUDE.md with execute_code RTK rules.

### 4. Set Up Auto-Prune Cron (if not exists)
```bash
# Create cron: daily at 3 AM WIB
hermes sessions prune --older-than 3 --source cron --yes
```
Schedule: `0 3 * * *` (daily 3 AM)
Deliver: `origin`

### 5. Optimize Memory
- Check memory usage (% of 2,200 char limit)
- Consolidate redundant entries
- Update RTK mandate entry if changed

### 6. Report Summary
Present combined analysis:
- Sessions: before → after pruning
- RTK coverage: commands filtered / total terminal calls
- Memory: usage %
- Auto-prune: active/paused

## Key Metrics to Track
| Metric | Target | How to Check |
|--------|--------|--------------|
| RTK coverage | 100% of eligible commands | `rtk gain` — compare count vs `hermes insights` tool calls |
| Session count | <100 (prune regularly) | `hermes sessions stats` |
| Memory usage | <90% | Memory tool response |
| Auto-prune | Active | `hermes cron list` |

## Common Issues
- **Low RTK coverage**: Patch CLAUDE.md with stricter execute_code rules
- **Memory at 100%**: Consolidate or remove entries using memory tool replace action
- **Session bloat**: Run prune for cron source first (safest to delete)
- **RTK name collision**: Check `which rtk` — should be Rust Token Kit, not Rust Type Kit
