---
name: camunda-bpmn-deployment-gotcha
description: Camunda BPMN 部署踩坑 — ENGINE-12018 historyTimeToLive 缺失问题
tags: [camunda, bpmn, spring-boot]
---

# Camunda BPMN 部署踩坑笔记

## 触发条件
使用 Camunda 7.21 + Spring Boot Starter 部署 BPMN XML 时，流程解析失败。

## 错误信息
```
ENGINE-09005 Could not parse BPMN process
ENGINE-12018 History Time To Live (TTL) cannot be null.
TTL is necessary for the History Cleanup to work.
```

## 根因
Camunda 7.x 默认要求每个流程定义必须设置 `historyTimeToLive`，否则拒绝解析 BPMN XML。

## 解决方案（3种）

### 方案1：BPMN XML 中直接标注（推荐）
```xml
<process id="my_proc" name="我的流程" isExecutable="true"
         camunda:historyTimeToLive="P30D">
```
命名空间需声明：
```xml
xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
```

### 方案2：application.yml 全局配置（不生效）
```yaml
camunda:
  bpm:
    historyTimeToLive: P30D  # camunda-bpm-spring-boot-starter 不读取此配置
```

### 方案3：禁用 TTL 强制检查（不推荐生产使用）
```yaml
camunda:
  bpm:
    enforceTTL: false
```

## 经验
- `historyTimeToLive` 写进 `application.yml` 对 `camunda-bpm-spring-boot-starter-rest` **不生效**
- 必须在 BPMN XML 的 `<process>` 标签上加 `camunda:historyTimeToLive="P30D"`
- Service 层可自动注入：用字符串替换 `isExecutable="true"` → `isExecutable="true" camunda:historyTimeToLive="P30D"`
- Camunda REST API `POST /deployment/create` multipart 上传时 Camunda 自动处理 TTL，不受此限制
