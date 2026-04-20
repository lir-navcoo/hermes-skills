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
| sources | daily, sessions, recall | 数据来源 |

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
- memory（长期记忆）
- daily（每日笔记）
- deep（深度梦境输出）

**执行配置**：
- speed: slow（慢速）
- thinking: high（深度思考）
- budget: expensive（高成本）

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

### 记忆来源
- **daily**: 每日笔记和工作记录
- **sessions**: 对话会话历史
- **recall**: 被主动回忆的记录
- **memory**: 已存储的长期记忆
- **logs**: 操作日志和行为记录

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
- REM梦境：模式分析+下周预测

## 参考
- OpenClaw源码：`src/memory-host-sdk/dreaming.ts`
- 触发条件：空闲30分钟无交互或定时
- 手动触发命令：`/dream`
