# 04 · 实时推送协议（WebSocket，后端 → 小程序）

> 用途：告警实时推送、设备在线状态、实时监测看板　|　局域网明文　|　版本 V1.6

## 1. 连接与鉴权

| 项 | 约定 |
|----|------|
| 端点 | `ws://{后端主机内网IP}:8080/medbox/ws?token={JWT}&topics=elder:e-1001`（局域网明文） |
| 鉴权 | 握手校验 JWT；按订阅主题（所监护老人 / 设备）鉴权，越权主题拒绝 |
| 心跳 | 客户端每 30s 发 `{"type":"ping"}`，服务端回 pong；超时断开 |
| 重连 | 断线指数退避重连（初始 1s，上限 30s）；重连后按 `lastEventId` 补推增量 |
| 补推窗口 | 服务端只保留**最近 200 条 / 最近 10 分钟**的事件用于补推，超出部分客户端应改用 REST 拉取全量 |
| 调试 | 用 `wscat -c "ws://{IP}:8080/medbox/ws?token=...&topics=elder:e-1001"` 验证；MQTT 侧用 EMQX Dashboard（见文档 03 第 5 章） |

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
| DEVICE_STATUS | 设备在线/离线（订阅 retained status） | deviceId, deviceType(`MAIN`/`TEMP_HUMI`/`LIGHT`/`CAMERA`), online, parentDeviceId（子设备时） |
| REMINDER | 临近服药计划提醒（可选，监护端提醒） | planId, planTime, **`medicines[]`**（一次提醒可含多种药） |
| AI_STREAM | AI 流式回答分片（**本期不使用**：流式统一走 HTTP SSE，见文档 05） | sessionId, delta |

> **DEVICE_STATUS 覆盖子设备**：药箱主控（`MAIN`）离线/上线各推一条；**传感器子设备**没有自己的 MQTT 连接，其在线状态由主控通过 `up/sensor/heartbeat` 汇总上报（见文档 03 第 3 章），后端据此推送子设备的 `DEVICE_STATUS`（带 `deviceType` 与 `parentDeviceId`）。子设备离线**不生成 DEVICE_OFFLINE 告警**，仅在设备详情页标注。

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
    "medicines":[ { "itemId":"pi-9001","medicineId":"m-205","medicineName":"阿莫西林","dose":"1","unit":"片" },
                  { "itemId":"pi-9002","medicineId":"m-388","medicineName":"维生素D","dose":"2","unit":"粒" } ] } }
```
