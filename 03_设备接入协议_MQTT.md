# 03 · 设备接入协议（MQTT，Broker = EMQX）

> 链路：设备 ⇄ EMQX ⇄ 后端　|　局域网内网部署　|　版本 V1.6

## 1. 连接与鉴权

| 项 | 约定 |
|----|------|
| Broker | **EMQX**，局域网内网部署：`tcp://{内网BrokerIP}:1883`（明文），详见文档 01 第 4 章。**设备**连这个地址；**后端**与 EMQX 同机时连 `localhost:1883`（各人本地一套环境，见文档 01 的 4.5） |
| clientId | = deviceId。**必填**（即使匿名连接也必须有）：EMQX 靠它区分身份、维护会话与遗嘱；相同 clientId 再次连接会踢掉前一个 |
| KeepAlive | 60s，心跳维持在线判定 |
| 遗嘱消息 LWT | topic=`.../status`，payload `{"online":false}`，异常掉线即时感知 |
| QoS | 遥测 QoS0/1；命令/事件 QoS1；关键下行命令 QoS1 且带 ack |
| Retained | status（在线状态）retained=true，订阅即得最新态 |
| 后端接入方式 | Java 后端以订阅方身份连到 EMQX（Spring Integration MQTT / Eclipse Paho），订阅 `.../up/#` 消费上报、向 `.../down/#` 发布下行 |
| EMQX 认证 | **本期不做认证**：EMQX 保持默认匿名（`allow_anonymous=true`），设备与后端能连到 1883 即可收发。将来需要时**分两级加固**：① 关闭匿名 + 共用用户名密码 + `clientId=deviceId` 白名单 + topic ACL；② 更强的**一机一密**（每台设备独立凭据 + EMQX HTTP 认证 / HMAC 动态签名 / 按 `${clientid}` 限制 topic）。详见文档 01 附录第 2 条 |
| 连接主体 | **一个药箱只有主设备（`MAIN`）建立 MQTT 连接**，clientId = 药箱 deviceId。温湿度 / 光照 / 摄像头等传感器是其**子设备**，不各自建连接，由主控在消息里带 `sensorId` 标明来源（见第 4 章） |
| 消息幂等 | **所有上行消息必须带 `msgId`**，格式固定为 **`{deviceId}-{10位递增序号}`**（如 `BOXA1001-0000000124`，序号持久化、重启不归零）。断线重连重发时服务端按 `msgId` 去重，避免重复写入服药记录 / 告警 |
| 下行确认 | 下行消息带 `msgId` 与 `needAck`；设备处理后向 `.../up/ack` 回执。**重发策略已确定：超时 5s 未回执则重发，最多 3 次**；3 次仍无回执则标记该下发失败并生成告警（见第 2 章） |

EMQX 自带 Dashboard（默认 `http://{内网IP}:18083`），可查看连接、Topic 监控与调试发布 / 订阅。

> **开发阶段不做认证的理由与底线**：每人一套本地 EMQX，端口只在局域网内；匿名模式下设备与后端都零配置，重建容器 / 换机器无需同步凭据。**但必须守住两条**：
>
> 1. **不要把 1883 / 18083 做端口映射或内网穿透到公网** —— 匿名 broker 一旦对外，任何人都能连上来发假数据、订阅所有 topic；
> 2. **EMQX Dashboard 的默认 `admin/public` 必须改** —— Dashboard 能查看所有消息、踢设备、改配置，默认密码等于裸奔（改一次仅需 1 分钟）。
>
> **唯一需要临时启用认证的场景**：在教室 / 公司等**公共 WiFi** 下演示 —— 同网段的人可直连你的 1883。此时要么临时启用认证，要么改用手机热点自己组网（更简单）。
>
> **关于认证的一个常识**：MQTT 协议本身不要求认证，`username` / `password` 是 CONNECT 报文里的**可选字段**；是否校验完全由 broker 策略决定。另外**只要匿名开着，clientId 白名单就不会生效**（匿名连接直接跳过认证器），想让白名单起作用必须先关闭匿名。

## 2. Topic 规范

统一前缀 **`medbox/{productKey}/{deviceId}/…`**（MQTT 规范不推荐以 `/` 开头，会产生空层级，故去掉前导斜杠），区分上行 up（设备发布）与下行 down（后端发布）。

| 方向 | Topic | QoS | 说明 |
|------|-------|-----|------|
| 设备→云 | `.../up/telemetry` | 0/1 | 环境遥测（**每个传感器各发一条**，只带自己负责的字段，见第 3 章） |
| 设备→云 | `.../up/event/dispense` | 1 | 服药事件（药品、剂量、时间、摄像头识别置信度；**拿错药也走这里**，用 `wrongDrug=true`） |
| 设备→云 | `.../up/event/error` | 1 | **设备侧异常**：门超时、识别失败、硬件故障等（不用于错服，见第 3 章口径说明） |
| 设备→云 | `.../up/sensor/heartbeat` | 0 | 主控周期性上报各传感器子设备在线状态（子设备无遗嘱，判离线靠它，见第 3 章） |
| 设备→云 | `.../up/inventory` | 1 | 库存变更（按药品的数量变化；生产日期与有效期由小程序录入，不从设备上报） |
| 设备→云 | **`.../up/ack`** | 1 | **下行消息回执**（计划 / 命令的 `msgId` 处理结果） |
| 设备→云 | `.../status (LWT)` | 0 | 在线/离线状态（retained） |
| 云→设备 | `.../down/schedule` | 1 | 下发/同步服药计划与提醒任务 |
| 云→设备 | `.../down/command` | 1 | 控制命令（蜂鸣/解锁/重启/校准） |

## 3. Payload 示例

> **上行消息统一带 `msgId`**，服务端按 `msgId` 幂等去重（同一 `msgId` 重复到达只处理一次）。

上行·环境遥测 `.../up/telemetry`（**每个传感器各发一条**，只带自己负责的字段）：

```json
// 温湿度传感器（TEMP_HUMI）
{ "msgId":"BOXA1001-000123", "ts":1759474487000,
  "sensorId":"BOXA1001-S-TH01",
  "temperature":24.6, "humidity":52.1 }

// 光照传感器（LIGHT）
{ "msgId":"BOXA1001-000124", "ts":1759474488000,
  "sensorId":"BOXA1001-S-LX01",
  "lux":120 }
```

`sensorId` = 该传感器子设备的 `device_id`（见文档 06 的 2.3）；主控转发时原样带上，后端据此区分来源、只更新对应列。

上行·服药事件 `.../up/event/dispense`（含错服判定与摄像头识别字段）：
```json
{ "msgId":"BOXA1001-000124", "ts":1759474500000,
  "planId":"p-3301", "planItemId":"pi-9001",
  "medicineId":"m-205",
  "actualDose":"1", "unit":"片", "wrongDrug":false, "onTime":true,
  "confidence":0.93, "source":"CAMERA", "imageId":"cap-8801" }
```

> 药箱**不划分仓位**，因此没有 `slotNo`：是否拿对药由摄像头**识别药品**判定（`medicineId` 是否等于计划药品），而不是比对格子。

字段说明：`confidence` = 摄像头识别置信度。**低于阈值时不要直接判错服**（会误报），抓拍标记为 `LOW_CONFIDENCE` 交由监护人人工复核；只有识别结果与计划药品**明确不符**才置 `wrongDrug=true`；`source` = `CAMERA` 摄像头识别 / `MANUAL` 手工补录；`imageId` = 本次服药的抓拍图片ID（对应 `capture.capture_id`）。**当前摄像头全量抓拍**，因此正常服药的 `imageId` 也不为空。

> **错服只走 `dispense`，不走 `error`**：两个通道的分工是 —— `up/event/dispense` = **服药事件**（无论拿对拿错，结果用 `wrongDrug` 表达，正常流程）；`up/event/error` = **设备侧异常**（门超时、识别失败、硬件故障），属于设备自己出问题、不是用户拿错药。这样后端只需在一个地方生成 WRONG_DRUG 告警，避免重复告警。

上行·设备异常 `.../up/event/error`（**仅设备侧异常**）：
```json
{ "msgId":"BOXA1001-000125", "ts":1759474512000, "type":"DOOR_TIMEOUT",
  "detail":"仓门开启超过 60s 未关闭" }
```
`type` 取值：`DOOR_TIMEOUT` 门超时 / `CAMERA_FAILED` 识别失败 / `HW_FAULT` 硬件故障。**错服用 `dispense.wrongDrug`，不要放这里。**

上行·库存变更 `.../up/inventory`（按药品，不按仓位）：
```json
{ "msgId":"BOXA1001-000127", "ts":1759474530000,
  "medicineId":"m-205", "quantity":12, "delta":-1 }
```
`quantity` = 变更后库存，`delta` = 本次变化量（服药 -1 / 补药 +n）。**生产日期与有效期不从设备上**，由小程序录入（见文档 02 第 5 章）。

上行·传感器心跳 `.../up/sensor/heartbeat`（主控周期性汇总子设备状态）：
```json
{ "msgId":"BOXA1001-000128", "ts":1759474540000,
  "sensors":[ { "sensorId":"BOXA1001-S-TH01","online":true },
              { "sensorId":"BOXA1001-S-LX01","online":true },
              { "sensorId":"BOXA1001-S-CAM01","online":false } ] }
```
因为**只有主控有 MQTT 连接**，子设备没有自己的遗嘱消息，故由主控每 **60s** 汇总上报一次；后端据此更新各子设备的 `last_heartbeat` 与 `online_state`。**离线判定（已确定）：连续 3 个周期（180s）未上报即判该传感器离线**，**不生成 DEVICE_OFFLINE 告警**，仅在设备详情与 `DEVICE_STATUS` 推送中标注。

上行·下行回执 `.../up/ack`（对应下行消息中的 `msgId`）：
```json
{ "msgId":"BOXA1001-000126", "ts":1759474520000,
  "ackFor":"d-001", "result":"OK", "reason":null }
```
`result`：`OK` / `FAILED`；`reason` 失败时给出简要原因（如 `"medicine_not_found"`）。服务端发出 `needAck:true` 的下行消息后，**5s 未收到回执则重发，最多 3 次**；仍无回执则标记下发失败并告警。

下行·同步计划 `.../down/schedule`（**一次提醒可含多种药，用 `items` 数组下发**）：
```json
{ "msgId":"d-001", "planId":"p-3301", "op":"UPSERT",
  "times":["08:00","20:00"], "repeat":"DAILY",
  "alarm":{"missAfterMin":15},
  "items":[ { "itemId":"pi-9001","medicineId":"m-205","dose":"1","unit":"片" },
            { "itemId":"pi-9002","medicineId":"m-388","dose":"2","unit":"粒" } ],
  "needAck":true }
```

> 设备按 `items` 逐种取药并**逐条上报**；本次提醒下所有 item 都上报才算完成，缺任一种药由后端判 MISS（见文档 06 的 2.6.1）。

## 4. 摄像头识别链路（WiFi → 单片机 → MQTT）

摄像头负责"取药是否正确 / 是否发生"的识别推理，**它既不直接连 EMQX 发 MQTT 消息**，也不参与业务判定，而是作为单片机的一个外设，通过 **WiFi** 与单片机通信：

```
摄像头(识别推理+抓拍) --WiFi--> 单片机(聚合 + 打时间戳) --MQTT up/event/dispense--> EMQX --> 后端(判定&入库)
摄像头(抓拍图片) ---------------------------------------- HTTP multipart -------> 后端(存储&过期清理)
```

> **摄像头侧的逻辑（含识别算法与抓拍策略）不由后端 / 小程序团队负责**，后端只负责"接收它给出的结果和图片"。**当前摄像头默认全量抓拍并上传：正常服药也拍、每次服药都拍**，后端按全量接收处理（存储与过期见文档 01 的 4.6、接口见文档 02 的 4.1）。

**为什么不让摄像头自己开一路 MQTT 直连后端**：

1. **两个时钟源**：`onTime` 是"取药时刻 vs 计划时间"的相对判定。蜂鸣提醒由单片机发出，识别结果由摄像头产生，若分开上报，后端要把两条消息按时间戳凑在一起，时钟漂移 / 网络抖动都会造成漏服误判；
2. **两条连接、两套鉴权**：clientId、连接凭据、遗嘱与在线状态都要做两份，排障困难；
3. **耦合算法迭代**：换摄像头模组 / 调识别算法都会波及后端协议。抽象成外设后，后端只见一个设备。

**职责划分**：

- 摄像头：只输出结构化结果（识别到的 `medicineId` / `confidence` / 时刻），**不输出图片给单片机**；
- 单片机：唯一 MQTT 客户端与时钟基准，知道"提醒何时触发"，把识别结果合成一条完整事件上报（不做判定，只做聚合与时间戳）；
- 后端：`onTime` 由后端按计划时间 + 容忍时长计算（与 MISS 判定口径一致），**改阈值无需重烧固件**；单片机本地算的 `onTime` 仅用于即时声光反馈，不作为入库依据。

**图片不走 MQTT**：MQTT 面向小负载，几百 KB 的 JPEG 会阻塞同连接上的其他消息且重传代价高。抓拍图片由摄像头用 **HTTP multipart 直传后端**（`POST /devices/{deviceId}/captures`，见文档 02 的 4.1），**不过单片机**（单片机既无带宽也无存储转发几百 KB 的图片）。

**对摄像头侧的要求（已确定）**：每个服药事件**只保留 1 张**代表性图片（禁止连拍多张上传）；抓拍时必须带 `capturedAt` 与 `confidence`，便于后端关联与复核。后端侧另有 `planId + 60s 时间窗`的兜底去重（见文档 02 的 4.1）。

**后续演进（本期不实现）**：若全量上传的带宽 / 存储压力不可接受，可在设备侧引入**边缘节点**（本地小主机或算力模组），由它先做数据清洗 —— 去重、丢弃模糊帧、只保留关键帧、必要时裁剪脱敏 —— 再把**结构化结果 + 精选图片**上传后端。届时后端接口与协议**无需改动**（仍是同一套 `up/event/dispense` 与 `POST /captures`），只是上游由摄像头换成边缘节点。

## 5. 联调与调试（没有真设备也能跑通链路）

设备固件未就绪时，用下面三种工具即可完成端到端验证。

### 5.1 EMQX Dashboard —— 主力工具（推荐）

访问 `http://{EMQX主机IP}:18083`（**首次登录后务必改掉默认 `admin/public`**）。

- **WebSocket 客户端**（Dashboard 内置工具）：可以**连接 + 订阅 + 发布**任意 topic，是联调 MQTT 最方便的方式，通常不需要再装 mosquitto；
- **连接 / 客户端管理**：确认设备是否在线、`clientId` 是否正确；
- **主题监控**：实时看消息有没有发出来、内容对不对。

典型用法：用 Dashboard 订阅 `medbox/{pk}/BOXA1001/up/#`，再手动发布一条 `up/event/dispense`，观察后端是否落 `med_record`、是否经 WebSocket 推送告警。

### 5.2 mosquitto_pub / mosquitto_sub —— 脚本造数据

`mosquitto_pub` **只能发布，不能订阅**；要看回包需另开终端用 `mosquitto_sub`。

```bash
# 模拟设备上报一次服药事件
mosquitto_pub -h 192.168.1.10 -p 1883 -i BOXA1001 -q 1 \
  -t 'medbox/pk/BOXA1001/up/event/dispense' \
  -m '{"msgId":"BOXA1001-0000000124","ts":1759474500000,"planId":"p-3301",
       "planItemId":"pi-9001","medicineId":"m-205","actualDose":"1","unit":"片",
       "wrongDrug":false,"onTime":true,"confidence":0.93,
       "source":"CAMERA","imageId":null}'

# 同时另开一个终端看上行是否到达
mosquitto_sub -h 192.168.1.10 -p 1883 -i debugger-01 \
  -t 'medbox/pk/BOXA1001/up/#' -v
```

注意 `-i` 的 clientId 必须等于 `deviceId`，否则后端按 clientId 找不到设备。

### 5.3 curl —— 后端 REST 与图片上传

见文档 02 的 1.4（含登录取 token、业务接口、multipart 上传抓拍图片的完整示例）。

### 5.4 WebSocket 推送

见文档 04（可用 `wscat` 或小程序端直连验证）。
