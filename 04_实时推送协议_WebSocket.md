# 04 · 实时推送协议（WebSocket，后端 → 小程序）

> 用途：告警实时推送、设备在线状态、实时监测看板　|　局域网明文　|　版本 V1.3

## 1. 连接与鉴权

| 项 | 约定 |
|----|------|
| 端点 | `ws://{后端主机内网IP}:8080/medbox/ws?token={JWT}&topics=elder:e-1001`（局域网明文） |
| 鉴权 | 握手校验 JWT；按订阅主题（所监护老人 / 设备）鉴权，越权主题拒绝 |
| 心跳 | 客户端每 30s 发 `{"type":"ping"}`，服务端回 pong；超时断开 |
| 重连 | 断线指数退避重连；重连后按 lastEventId 补推增量 |

## 2. 消息帧结构

```json
{ "type":"ALARM|DEVICE_STATUS|REMINDER|AI_STREAM|PONG",
  "id":"evt-9a1c",           // 单调递增/雪花，用于断线补推
  "topic":"elder:e-1001",
  "ts":1759474520000,
  "payload": { } }
```

## 3. 推送事件类型

| type | 触发时机 | payload 要点 |
|------|----------|--------------|
| ALARM | 后端生成漏服/错服/过期/环境告警 | alarmId, type, level, message, elderId |
| DEVICE_STATUS | 设备在线/离线（订阅 retained status） | deviceId, online, battery |
| REMINDER | 临近服药计划提醒（可选，监护端提醒） | planId, planTime, medicineName |
| AI_STREAM | AI 流式回答分片（备选传输通道） | sessionId, delta |

**示例：推送漏服告警**
```json
{ "type":"ALARM","id":"evt-9a1c","topic":"elder:e-1001",
  "ts":1759474520000,
  "payload":{ "alarmId":"a-5001","type":"MISS","level":"HIGH",
    "message":"阿莫西林 08:00 计划服药未按时服用","medicineName":"阿莫西林" } }
```
