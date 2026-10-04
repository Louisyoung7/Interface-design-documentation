# 04 · 实时推送协议（WebSocket，后端 → 小程序）

> 用途：告警实时推送、设备在线状态、实时监测看板　|　局域网明文　|　版本 V1.4

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
| ALARM | 后端生成漏服/错服/过期/环境告警 | alarmId, type, level(`INFO`/`WARN`/`CRITICAL`), message, elderId, medicineId |
| DEVICE_STATUS | 设备在线/离线（订阅 retained status） | deviceId, online |
| REMINDER | 临近服药计划提醒（可选，监护端提醒） | planId, planTime, **`medicines[]`**（一次提醒可含多种药） |
| AI_STREAM | AI 流式回答分片（备选传输通道） | sessionId, delta |

**示例：推送漏服告警**
```json
{ "type":"ALARM","id":"evt-9a1c","topic":"elder:e-1001",
  "ts":1759474520000,
  "payload":{ "alarmId":"a-5001","type":"MISS","level":"WARN",
    "message":"08:00 计划服药未按时服用：缺 阿莫西林、维生素D",
    "medicineIds":["m-205","m-388"] } }
```

**示例：推送服药提醒（多种药）**
```json
{ "type":"REMINDER","id":"evt-9a1d","topic":"elder:e-1001",
  "ts":1759474500000,
  "payload":{ "planId":"p-3301","planTime":"2026-10-04T00:00:00Z",
    "medicines":[ { "itemId":"pi-9001","medicineId":"m-205","medicineName":"阿莫西林","dose":"1","unit":"片","slotNo":3 },
                  { "itemId":"pi-9002","medicineId":"m-388","medicineName":"维生素D","dose":"2","unit":"粒","slotNo":5 } ] } }
```
