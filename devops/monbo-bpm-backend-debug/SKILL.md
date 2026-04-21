---
name: monbo-bpm-backend-debug
description: monbo-bpm 后端调试修复记录 — JDK/Lombok兼容、Camunda API调用Bug、CORS配置
tags: [monbo-bpm, camunda, java, spring-boot]
---

# monbo-bpm Backend Debugging & Fixes

## JDK + Lombok 兼容性

**症状**：`Fatal error compiling: java.lang.ExceptionInInitializerError: com.sun.tools.javac.code.TypeTag :: UNKNOWN`

**原因**：JDK 25 + Lombok 1.18.36 不兼容

**修复**：降级 Lombok 至 1.18.32

```xml
<lombok.version>1.18.32</lombok.version>
```

**本机 JDK 路径**：`/opt/homebrew/Cellar/openjdk@21/21.0.10/libexec/openjdk.jdk/Contents/Home`

编译命令：
```bash
export JAVA_HOME=$(/usr/libexec/java_home -F --failfast 2>/dev/null || echo "/opt/homebrew/Cellar/openjdk@21/21.0.10/libexec/openjdk.jdk/Contents/Home")
cd /Users/lirui/monbo-bpm/monbo-bpm-api && mvn compile -q
```

---

## Bug 1: activate/suspend 流程定义ID格式错误

**文件**：`ProcessDefServiceImpl.java` 第219、239行

**问题**：传给 Camunda 的是 `key:version`，但 Camunda API 需要完整 ID `key:version:deploymentId`

**修复**：
```java
// 错误
String camundaDefId = def.getProcessKey() + ":" + def.getVersion();
repositoryService.activateProcessDefinitionById(camundaDefId, ...);

// 正确：使用 entity 中已存储的完整 ID
repositoryService.activateProcessDefinitionById(def.getCamundaProcessDefId(), ...);
```

---

## Bug 2: 流程实例状态不同步

**文件**：`ProcessInst.java`、`ProcessInstServiceImpl.java`

**问题**：Camunda 侧实例结束（正常完成/取消），本地 `status` 永远是 1

**修复**：
1. `ProcessInst` 实体新增 `endedTime datetime` 字段
2. DDL：`ALTER TABLE mb_process_inst ADD COLUMN ended_time datetime DEFAULT NULL`
3. `cancelProcessInst` 写入 `endedTime`
4. 新增 `syncFromCamunda()` 方法：先查 RuntimeService，不在运行时查 HistoryService 确认结束状态
5. `getProcessInstById` 和 `listByProcessDef` 查询前触发同步

---

## CORS 配置

**文件**：`SecurityConfig.java`

`CorsConfigurationSource` Bean：允许所有来源、全部方法、全部头、credentials、maxAge=3600。在 `filterChain` 中通过 `.cors()` 启用。
