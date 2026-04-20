---
name: astock-vault
description: 基于 Obsidian vault 的 A 股分析知识库初始化与维护流程
version: 1.0.0
tags: [Obsidian, 知识库, A股, 笔记系统]
---

# A股分析 Obsidian Vault

## 概述

用 Obsidian 作为 A 股分析知识库，记录每日选股、复盘、持仓追踪。

Vault 路径：`~/Documents/Obsidian Vault/`

## 目录结构

```
A股分析/
├── INDEX.md                    # 主索引（含持仓一览、近期更新）
├── 早盘推荐/
│   └── INDEX.md                # 早盘索引
│   └── YYYY-MM-DD-早盘推荐.md  # 每日早盘选股报告
├── 下午复盘/
│   └── INDEX.md                # 复盘索引
│   └── YYYY-MM-DD-复盘.md     # 每日盘后复盘
├── 标的追踪/
│   └── INDEX.md                # 持仓索引
│   └── 代码-名称.md            # 自有持仓（如 002097-山河智能.md）
└── 市场洞察/
    └── INDEX.md                # 市场洞察索引
```

## 持仓文件格式

每只持仓一个 md 文件，frontmatter 包含持仓成本和数量：

```yaml
---
title: 002097 山河智能
created: 2026-04-19
updated: 2026-04-19
type: 持仓标的
tags: [持仓, 002097, 山河智能]
cost: 24.252
shares: 100
---
```

## INDEX.md 格式

每个目录的 INDEX.md 包含该目录的文件清单和元数据。

## 执行时机

- **cron 早盘任务** → 写入 `早盘推荐/YYYY-MM-DD-早盘推荐.md`
- **cron 复盘任务** → 写入 `下午复盘/YYYY-MM-DD-复盘.md`
- **持仓更新** → 直接修改对应 `标的追踪/代码-名称.md`
