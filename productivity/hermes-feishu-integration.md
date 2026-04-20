---
name: hermes-feishu-integration
description: 飞书（Feishu/Lark）机器人接入Hermes Agent的完整流程，包含环境配置、launchd注意事项、权限设置和常见问题排查
version: 1.0.0
tags: [飞书, Feishu, Hermes, 集成, 机器人]
---

# Hermes + 飞书接入指南

## 核心要点

飞书已在Hermes中原生集成（`gateway/platforms/feishu.py`），但macOS上需注意launchd服务的环境变量问题。

---

## 一、前置准备

### 1. 创建飞书应用
- 在 https://open.feishu.cn/app 创建应用
- 获取 `App ID`（格式：`cli_xxx`）和 `App Secret`
- 开启「机器人」能力
- 配置权限：需要 `im:message`、`im:message:receive` 等

### 2. 安装依赖
```bash
pip3 install lark-oapi
```

---

## 二、环境变量配置（关键！）

**macOS上launchd服务不继承shell环境变量**，因此不能只写`.env`，必须直接写在plist文件里。

### 步骤

1. 编辑 launchd plist：
```bash
vim ~/Library/LaunchAgents/ai.hermes.gateway.plist
```

2. 在 `EnvironmentVariables` dict 中添加：
```xml
<key>FEISHU_APP_ID</key>
<string>cli_你的AppID</string>
<key>FEISHU_APP_SECRET</key>
<string>你的AppSecret</string>
<key>FEISHU_CONNECTION_MODE</key>
<string>websocket</string>
<key>FEISHU_HOME_CHANNEL</key>
<string>机器人的open_id</string>
<key>FEISHU_HOME_CHANNEL_NAME</key>
<string>多宝</string>
<key>FEISHU_ALLOWED_USERS</key>
<string></string>
<key>GATEWAY_ALLOW_ALL_USERS</key>
<string>true</string>
```

3. 重新加载服务：
```bash
launchctl unload ~/Library/LaunchAgents/ai.hermes.gateway.plist
launchctl load ~/Library/LaunchAgents/ai.hermes.gateway.plist
```

---

## 三、验证连接状态

```bash
# 查看gateway日志
tail -f ~/.hermes/logs/gateway.log

# 检查飞书连接（应该有 [Lark] connected to wss://msg-frontier.feishu.cn ）
grep "Lark.*connected" ~/.hermes/logs/gateway.log

# 检查机器人状态
curl -s -X GET 'https://open.feishu.cn/open-apis/bot/v3/info' \
  -H 'Authorization: Bearer TOKEN'
# activate_status: 0=未激活, 1=未知, 2=被禁用, 3=正常
```

---

## 四、常见问题

### 1. "No messaging platforms enabled"
- 原因：launchd服务没有读取到FEISHU环境变量
- 解决：环境变量必须写在plist的EnvironmentVariables里，不能只靠.env

### 2. "lark-oapi not installed"
- 原因：lark-oapi包未安装，或安装在系统Python而非venv
- 解决：`pip3 install lark-oapi`

### 3. activate_status=0 或 2
- 0 = 机器人未激活（需要在飞书开放平台启用）
- 2 = 机器人被禁用（需要在应用管理后台启用）
- 解决：在 https://open.feishu.cn/app 对应应用下开启机器人能力

### 4. "Unable to hydrate bot identity"
- 警告级别，不影响基本功能
- 原因：缺少 `admin:app.info:readonly` 或 `application:application:self_manage` 权限
- 解决：在飞书开放平台给应用添加这些权限

### 5. 机器人不回复消息
- 检查 `GATEWAY_ALLOW_ALL_USERS=true` 是否设置
- 检查飞书Bot是否已添加到会话
- 查看日志确认WS连接是否建立

---

## 五、发送测试消息

在飞书客户端找到机器人，直接发消息测试。响应说明接入成功。

---

## 六、Cron任务发送到飞书

创建cron任务时设置 `deliver="feishu:open_id"` 即可推送到飞书。
