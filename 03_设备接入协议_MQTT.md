# 03 · 设备接入协议（MQTT，Broker = EMQX）

> 链路：设备 ⇄ EMQX ⇄ 后端　|　局域网内网部署　|　版本 V1.4

## 1. 连接与鉴权

| 项 | 约定 |
|----|------|
| Broker | **EMQX**，局域网内网部署：`tcp://{内网BrokerIP}:1883`（明文），详见文档 01 第 4 章。**设备**连这个地址；**后端**与 EMQX 同机时连 `localhost:1883`（各人本地一套环境，见文档 01 的 4.5） |
| clientId | = deviceId，凭 MQTT 用户名密码或一机一密（DeviceSecret+签名/token）接入，由 HTTP 注册接口下发 |
| KeepAlive | 60s，心跳维持在线判定 |
| 遗嘱消息 LWT | topic=`.../status`，payload `{"online":false}`，异常掉线即时感知 |
| QoS | 遥测 QoS0/1；命令/事件 QoS1；关键下行命令 QoS1 且带 ack |
| Retained | status（在线状态）retained=true，订阅即得最新态 |
| 后端接入方式 | Java 后端以订阅方身份连到 EMQX（Spring Integration MQTT / Eclipse Paho），订阅 `.../up/#` 消费上报、向 `.../down/#` 发布下行 |
| EMQX 认证 | 局域网阶段用内置数据库 / HTTP 认证或用户名密码 + clientId(=deviceId) 白名单 |
| 消息幂等 | **所有上行消息必须带 `msgId`**（设备侧生成，建议 `deviceId+序号/时间戳`）。断线重连重发时服务端按 `msgId` 去重，避免重复写入服药记录 / 告警 |
| 下行确认 | 下行消息带 `msgId` 与 `needAck`；设备处理后向 `.../up/ack` 回执，超时未回执由服务端重发（见第 2 章） |

EMQX 自带 Dashboard（默认 `http://{内网IP}:18083`），可查看连接、Topic 监控与调试发布 / 订阅。

## 2. Topic 规范

统一前缀 **`medbox/{productKey}/{deviceId}/…`**（MQTT 规范不推荐以 `/` 开头，会产生空层级，故去掉前导斜杠），区分上行 up（设备发布）与下行 down（后端发布）。

| 方向 | Topic | QoS | 说明 |
|------|-------|-----|------|
| 设备→云 | `.../up/telemetry` | 0/1 | 环境遥测（温湿度光照，周期上报） |
| 设备→云 | `.../up/event/dispense` | 1 | 取药/服药事件（仓位、剂量、时间） |
| 设备→云 | `.../up/event/error` | 1 | 错服、门超时、硬件故障等异常事件 |
| 设备→云 | `.../up/inventory` | 1 | 仓位库存/有效期变更（RFID 变化） |
| 设备→云 | **`.../up/ack`** | 1 | **下行消息回执**（计划 / 命令的 `msgId` 处理结果） |
| 设备→云 | `.../status (LWT)` | 0 | 在线/离线状态（retained） |
| 云→设备 | `.../down/schedule` | 1 | 下发/同步服药计划与提醒任务 |
| 云→设备 | `.../down/command` | 1 | 控制命令（蜂鸣/解锁/重启/校准） |

## 3. Payload 示例

> **上行消息统一带 `msgId`**，服务端按 `msgId` 幂等去重（同一 `msgId` 重复到达只处理一次）。

上行·环境遥测 `.../up/telemetry`：
```json
{ "msgId":"BOXA1001-000123", "ts":1759474487000,
  "temperature":24.6, "humidity":52.1, "lux":120 }
```

上行·服药事件 `.../up/event/dispense`（含错服判定字段）：
```json
{ "msgId":"BOXA1001-000124", "ts":1759474500000,
  "planId":"p-3301", "planItemId":"pi-9001",
  "slotNo":3, "medicineId":"m-205",
  "actualDose":"1", "unit":"片", "wrongDrug":false, "onTime":true }
```

上行·错服异常 `.../up/event/error`：
```json
{ "msgId":"BOXA1001-000125", "ts":1759474512000, "type":"WRONG_DRUG",
  "expectedSlot":3, "actualSlot":5,
  "expectedMedicine":"m-205", "actualMedicine":"m-388" }
```

上行·下行回执 `.../up/ack`（对应下行消息中的 `msgId`）：
```json
{ "msgId":"BOXA1001-000126", "ts":1759474520000,
  "ackFor":"d-001", "result":"OK", "reason":null }
```
`result`：`OK` / `FAILED`；`reason` 失败时给出简要原因（如 `"slot_not_found"`）。服务端发出 `needAck:true` 的下行消息后若超时未收到回执，按策略重发（关键命令最多重发 N 次，超出则标记下发失败并告警）。

下行·同步计划 `.../down/schedule`（**一次提醒可含多种药，用 `items` 数组下发**）：
```json
{ "msgId":"d-001", "planId":"p-3301", "op":"UPSERT",
  "times":["08:00","20:00"], "repeat":"DAILY",
  "alarm":{"missAfterMin":15},
  "items":[ { "itemId":"pi-9001","medicineId":"m-205","slotNo":3,"dose":"1","unit":"片" },
            { "itemId":"pi-9002","medicineId":"m-388","slotNo":5,"dose":"2","unit":"粒" } ],
  "needAck":true }
```

> 设备按 `items` 逐仓取药并**逐条上报**；本次提醒下所有 item 都上报才算完成，缺任一种药由后端判 MISS（见文档 06 的 2.6.1）。
