---
name: camunda-bpmn-integration-gotcha
description: Camunda BPMN 引擎集成踩坑记录 — TTL设置、激活/挂起API、Starter自动配置、BPMN-JSON序列化
---

# Camunda BPMN 引擎集成踩坑

## 环境
- Camunda 7.21.0 (camunda-bpm-spring-boot-starter-rest)
- Spring Boot 3.2.5 + Java 21
- MySQL

---

## 1. BPMN XML 必须设置 historyTimeToLive（ENGINE-12018）

**问题：** 部署 BPMN XML 时报错：
```
ENGINE-09005 Could not parse BPMN process.
ENGINE-12018 History Time To Live (TTL) cannot be null.
```

**原因：** Camunda 7.21 强制要求每个流程定义设置 TTL。

**错误尝试（无效）：**
```yaml
# application.yml 中配置不生效
camunda:
  bpm:
    historyTimeToLive: P30D  # ❌ 引擎配置层不生效
```

**正确方案：** BPMN XML 中必须设置：
```xml
<process id="my_process" name="我的流程" isExecutable="true"
         camunda:historyTimeToLive="P30D"
         xmlns:camunda="http://camunda.org/schema/1.0/bpmn">
```

---

## 2. 激活/挂起用 byId 而非 byKey（影响范围）

**问题：** `repositoryService.activateProcessDefinitionByKey(key, true, null)` 会激活该 key **所有版本**。

**正确方案：** 用 `key:version` 格式指定特定版本：
```java
String camundaDefId = def.getProcessKey() + ":" + def.getVersion();
repositoryService.activateProcessDefinitionById(camundaDefId, true, null);
repositoryService.suspendProcessDefinitionById(camundaDefId, true, null);
```

---

## 3. Spring Boot Starter 自动配置引擎，不要自定义 ProcessEngine 等 Bean

**问题：** 自定义 `@Bean ProcessEngine` / `RepositoryService` / `RuntimeService` 导致冲突。

**原因：** `camunda-bpm-spring-boot-starter-rest` 已自动配置这些 Bean。

**正确方案：** 直接注入即可，无需任何配置类：
```java
@Service
public class MyService {
    private final RepositoryService repositoryService;
    // Starter 已自动配置，直接注入
}
```

---

## 4. BPMN XML 含特殊字符时 JSON 序列化

**问题：** 通过 API 传输 BPMN XML（含 `< > &` 等），shell heredoc 会丢失或转义错误。

**正确方案：** 使用 Python json.dumps() 正确序列化：
```python
import json
bpmn = '<?xml version="1.0"?><process id="p1">...</process>'
payload = json.dumps({'bpmnXml': bpmn})  # 正确转义
```

---

## 5. Camunda REST API 部署（multipart/form-data）

通过 REST API 直接部署 BPMN（绕过 Java API）：
```python
boundary = '----BPMNBoundary'
body = (
    f'--{boundary}\r\n'
    f'Content-Disposition: form-data; name="data"; filename="process.bpmn"\r\n'
    f'Content-Type: application/xml\r\n\r\n'
).encode('utf-8') + bpmn_xml_bytes + f'\r\n--{boundary}--\r\n'.encode('utf-8')
# Content-Type: multipart/form-data; boundary=----
```

Camunda REST API 地址：`http://localhost:8080/engine-rest/deployment/create`

---

## 验证清单

部署新 BPMN 后验证：
1. BPMN XML 中含 `camunda:historyTimeToLive="P30D"` ✓
2. 激活/挂起用 `byId(key:version)` ✓
3. 无自定义 ProcessEngine Bean 冲突 ✓
