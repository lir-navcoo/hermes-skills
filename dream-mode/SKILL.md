---
name: dream-mode
description: OpenClaw Dream Mode周期性记忆整合机制 - 三级记忆等级系统
trigger: 空闲30分钟无交互或定时触发
---

# Dream Mode - 周期性记忆整合与反思学习

## 概述
Dream Mode是OpenClaw的周期性记忆整合机制，模拟人类睡眠中的记忆整理过程。核心是**三级记忆等级系统**，对不同重要度的记忆进行分层处理。

## 记忆等级体系

### 1. Light Dreaming（轻度梦境）
**用途**：日常快速整合，轻量级记忆刷新

| 参数 | 默认值 | 说明 |
|------|--------|------|
| cron | `0 */6 * * *` | 每6小时执行 |
| lookbackDays | 2 | 回溯2天 |
| limit | 100 | 最多处理100条 |
| dedupeSimilarity | 0.9 | 相似度≥0.9去重 |
| sources | sessions, daily, recall, memory, logs | 数据来源（全部纳入） |

**执行配置**：
- speed: fast（快速）
- thinking: low（低思考深度）
- budget: cheap（低成本）

### 2. Deep Dreaming（深度梦境）
**用途**：重要记忆的深度整合与长期保存

| 参数 | 默认值 | 说明 |
|------|--------|------|
| cron | `0 3 * * *` | 每天凌晨3点 |
| limit | 10 | 每次最多10条 |
| minScore | 0.8 | 最低评分阈值 |
| minRecallCount | 3 | 最少被回忆次数 |
| minUniqueQueries | 3 | 最少独立查询数 |
| recencyHalfLifeDays | 14 | 新近度半衰期（天） |
| maxAgeDays | 30 | 最大记忆年龄 |

**评分算法**：
```
score = recency_weight × recall_count × unique_queries × base_score
recency_weight = 0.5 ^ (days_since_last_recall / 14)
```

**恢复机制**：
- 当健康度 < 0.35 时触发
- 回溯30天内的失败记忆
- 最多20个候选，自动写入置信度≥0.97

**执行配置**：
- speed: balanced（平衡）
- thinking: high（深度思考）
- budget: medium（中等成本）

### 3. REM Dreaming（REM梦境）
**用途**：识别跨记忆的行为模式，类比人类REM睡眠

| 参数 | 默认值 | 说明 |
|------|--------|------|
| cron | `0 5 * * 0` | 每周日凌晨5点 |
| lookbackDays | 7 | 回溯7天 |
| limit | 10 | 最多10个模式 |
| minPatternStrength | 0.75 | 最小模式强度 |

**数据来源**：
- memory（Hermes 持久记忆）
- sessions（会话历史）
- daily（Obsidian vault）
- logs（操作日志）

**执行配置**：
- speed: slow（慢速）
- thinking: high（深度思考）
- budget: expensive（高成本）

#### Skills 优化分析（REM 中嵌入）

基于 memory、sessions、daily、logs 全部数据，每周分析 skills 使用状况：

**分析维度**：
- **使用频率**：哪些 skills 被频繁调用，哪些从未使用或长期未调用
- **失效检测**：API 变更、技术栈升级导致 skills 过时或描述错误
- **缺失发现**：BOSS 需求与现有 skills 之间的 Gap，需求反复出现但无 skill 支持
- **踩坑记录**：sessions 中反复出现的同类错误，是否需要新增/优化 skill 避免重复踩坑
- **记忆反馈**：BOSS 纠正过的方法是否已同步到对应 skill，memory 中的项目规范是否与 skills 一致
- **过时清理**：超过 30 天未调用的 skills，评估是否应归档或移除

**Skills 优化报告格式**（整合进 REM 梦境报告）：

```
## Skills 优化报告 — {date}

### 🔴 需要立即修复（影响日常使用）
- {skill名}：{问题描述} → {建议修复方案}
  - 依据：{来源} — {具体引用}

### 🟡 建议优化（提升效率）
- {skill名}：{优化点} → {建议方案}
  - 依据：{来源} — {具体引用}

### 🟢 新增建议（基于需求分析）
- {需求描述} → 建议新增「{skill名}」，核心功能：{功能列表}
  - 依据：{sessions/daily 中的具体需求}

### 📌 优化依据来源
| 类型 | 记忆内容摘要 |
|------|------------|
| memory | {关键项目规范/技术决策} |
| sessions | {本周期内的开发任务/问题/决策} |
| daily | {Obsidian 中的工作记录} |
```

**不活跃 Skills 检测（每月一次）**：
- 从 sessions 日志统计各 skill 调用频率
- 超过 30 天未调用 → 列入「建议审查」列表
- 若 skill 描述的技术/工具已过时 → 建议移除或归档

---

## 核心机制

### 评分与筛选
```
候选记忆评分 = 
  base_score × 
  recency_decay(days_since_access, half_life=14) × 
  recall_frequency_weight × 
  uniqueness_factor
```

### 健康度指标
```
health = (active_memories / max_capacity) × pattern_coherence
```
- health < 0.35 → 触发恢复机制
- health 0.35-0.7 → 正常深度梦境
- health > 0.7 → 轻度整合即可

## 记忆来源

所有记忆类型均纳入整合：

| 来源 | 说明 | 存储位置 |
|------|------|----------|
| `sessions` | 对话会话历史 | Hermes 长期记忆 |
| `daily` | 每日笔记和工作记录 | Obsidian vault |
| `recall` | 被主动回忆的记录 | Hermes 长期记忆 |
| `memory` | Hermes 持久化记忆（memory 工具） | ~/.hermes/memory.json |
| `logs` | 操作日志和行为记录 | Hermes 工作目录 |

### 各来源数据特征

- **sessions**：开发任务进度、代码决策、问题解决过程
- **daily**：Obsidian vault 中的工作记录（A股分析/开发笔记/项目复盘）
- **recall**：跨 session 关键上下文（用户偏好、项目约定、工具踩坑）
- **memory**：Hermes 持久化记忆（项目结构、技术栈、评审规范）
- **logs**：git 操作、文件变更、服务启停记录

### 整合优先级

1. **memory**（Hermes 持久记忆）— 项目约定、技术规范、用户偏好
2. **sessions**（会话历史）— 开发决策、问题解决、任务进度
3. **daily**（Obsidian vault）— A股分析笔记、项目文档
4. **recall**（回忆记录）— 被反复引用的上下文
5. **logs**（操作日志）— 辅助验证行为模式

## 安全约束
- 只读模式，不执行外部操作
- 不修改原始记忆内容
- 记忆输出仅用于反思和优化

## 配置示例
```yaml
dreaming:
  enabled: true
  frequency: "0 3 * * *"
  timezone: "Asia/Shanghai"
  storage:
    mode: "separate"
    separateReports: true
  phases:
    light:
      enabled: true
      lookbackDays: 2
      limit: 100
    deep:
      enabled: true
      minScore: 0.8
      minRecallCount: 3
      recencyHalfLifeDays: 14
    rem:
      enabled: true
      lookbackDays: 7
      minPatternStrength: 0.75
```

## 在Hermes Agent中实现

### Cron任务设计
1. **轻度梦境** - 每6小时（`0 */6 * * *`）
2. **深度梦境** - 每天凌晨3点（`0 3 * * *`）
3. **REM梦境** - 每周日凌晨5点（`0 5 * * 0`）

### 执行流程
1. 扫描对应时间范围内的记忆/笔记
2. 按评分算法排序
3. 执行对应深度的整合操作
4. 生成梦境报告
5. 邮件通知BOSS

### 邮件通知
- 轻度梦境：简单汇总
- 深度梦境：详细报告+优化建议
- REM梦境：模式分析+Skills优化报告+下周预测

## 参考
- OpenClaw源码：`src/memory-host-sdk/dreaming.ts`
- 触发条件：空闲30分钟无交互或定时
- 手动触发命令：`/dream`
