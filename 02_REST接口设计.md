# 02 · REST 接口设计

> 所有路径均相对 Base URL：`http://{后端主机内网IP}:8080/medbox/api/v1`　|　版本 V1.4
> 标注【鉴权】表示需携带 JWT，监护数据类接口需校验权限。AI 相关接口见文档 05。

## 1. 通用约定

| 项 | 约定 |
|----|------|
| Base URL | `http://{后端主机内网IP}:8080/medbox/api/v1`（局域网内网地址，见文档 01） |
| 请求体格式 | `application/json; charset=utf-8` |
| 认证方式 | Header: `Authorization: Bearer {JWT}`（设备端走 MQTT 用户名密码 / 一机一密，不走此通道） |
| 时间格式 | ISO-8601 **UTC**，如 `2026-10-03T07:04:00Z`；前端本地化展示 |
| 时区口径 | 除服药计划的 `times`（见第 6 章）外，**所有时间一律 UTC**。`med_plan.times` 存**设备本地墙钟时间**（如 `"08:00"`），时区取 `device.timezone`（默认 `Asia/Shanghai`）；服务端判定时换算成 UTC 后再比较 |
| 幂等 | 写操作支持 Header: `X-Request-Id`（UUID）去重 |
| 分页 | query 参数 `page`(从 1 起)、`size`(默认 20, 上限 100)；响应返回 `total` |
| 版本 | URI 版本化 `/v1`；向后兼容字段追加不升版本 |

### 1.1 统一响应结构

```json
{
  "code": 0,            // 0=成功，非0=业务错误码
  "message": "success",
  "data": { },          // 业务数据；失败时可为 null
  "traceId": "9f2c...", // 链路追踪，便于排障
  "timestamp": 1759474487
}
```

### 1.2 通用错误码

| code | HTTP | 含义 |
|------|------|------|
| 0 | 200 | 成功 |
| 40001 | 400 | 参数校验失败 |
| 40101 | 401 | 未登录 / Token 过期 |
| 40102 | 401 | 账号或密码错误 |
| 40301 | 403 | 无权限（越权访问他人数据 / 设备） |
| 40401 | 404 | 资源不存在 |
| 40302 | 403 | 监护关系未生效（关系仍为 PENDING，老人端尚未确认） |
| 40901 | 409 | 状态冲突（如设备离线无法下发命令） |
| 42901 | 429 | 触发限流 |
| 50000 | 500 | 服务端内部错误 |
| 50310 | 503 | AI 大模型服务暂不可用 / API Key 无效 |

### 1.3 权限默认规则

除接口上显式标注【鉴权·仅监护人】等限定外，按下述默认规则校验（角色定义见文档 01 第 3 章）：

| 角色 | 默认权限 |
|------|----------|
| 老人本人 | **只读本人数据**：本人计划、记录、告警、本人设备状态、AI 问答；不可改计划 / 配置 Key |
| 监护人 / 子女 | 对**已生效（ACTIVE）监护关系**下的老人：**可读可写**（计划、药品、库存、告警处理、阈值、AI Key 配置） |
| 社区护理人员 | 所辖老人数据**只读** + **处理告警** + **导出记录**；不可改计划与药品 |
| 家庭医生 | **制定 / 修改服药计划** + **维护药品禁忌知识库** + 查看依从性报告；不处理告警导出 |
| 设备端 | 仅 MQTT 通道（不走 REST，见文档 03） |

**通用校验顺序**：① 是否登录（40101）→ ② 对目标 `elderId` / `deviceId` 是否有监护 / 管理关系（40301）→ ③ 关系是否已生效（40302）→ ④ 该角色是否具备此操作权限（40301）。越权一律 40301，不区分"资源不存在"与"无权限"，避免资源枚举。

## 2. 核心数据模型

> 全部实体清单与建表 DDL 见文档 06《数据库设计》。本文件只列接口用到的实体名，便于对照路径含义。

涉及实体：User（老人/监护人）、GuardianRelation、Device（药箱主控 + 传感器子设备）、Medicine、MedicineStock（库存）、MedPlan（服药计划）、MedRecord（服药记录）、EnvSample（环境遥测）、Alarm（告警）、AlarmSetting（告警规则）、LlmConfig（大模型配置）、DrugManual（说明书）、DrugManualChunk（说明书向量分块，见文档 05 / 06）。


## 3. 认证与用户

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /auth/register | 账号密码注册（手机号 + 密码 + 角色），返回 JWT |
| POST | /auth/login | **账号密码登录**（手机号 / 用户名 + 密码），返回 JWT |
| POST | /auth/refresh | 刷新 Token |
| GET | /users/me | 【鉴权】当前用户资料与角色 |
| POST | /users/{elderId}/guardians | 【鉴权·监护人发起】申请绑定监护关系（创建 `PENDING` 关系） |
| GET | /users/guardians/pending | 【鉴权·老人端】我待确认的监护请求列表 |
| PATCH | /users/guardians/{relationId}/accept | 【鉴权·老人端】确认监护请求（置 `ACTIVE`） |
| DELETE | /users/guardians/{relationId} | 【鉴权】拒绝 / 解除监护关系（老人端拒绝、任一方解除） |
| GET | /users/{elderId}/profile | 【鉴权】老人基础信息（监护人可见） |

> **监护绑定为两步**：① 监护人发起申请 → 关系 `PENDING`；② 老人端在"待确认"列表中确认 → 关系 `ACTIVE`，监护人才真正获得读写权限。未确认前访问该老人数据返回 **40302**。

> **认证方式约定**：小程序端主登录方式为**账号 + 密码**，**不再使用短信验证码**（免去短信网关、验证码下发与存储）。
>
> - 登录账号为手机号或用户名；密码服务端 **BCrypt 加盐哈希**存储，库内不存明文，登录失败统一返回 `40102` 不区分"账号不存在 / 密码错误"；
> - 局域网阶段为明文 HTTP，密码在传输层可见；在意的话前端可先做一次 SHA-256 再传（**不是安全替代**），正式环境直接上 HTTPS（见文档 01 附录）；
> - **本阶段不做微信登录**：`wx.login` + `code2session` 虽然是免费的基础能力（个人主体小程序即可用，无需微信认证），但仍需后端出网访问 `api.weixin.qq.com`、配置 appid / secret，并额外设计"首次微信登录如何绑定已有账号"的流程。当前统一走账号密码；后续需要时再加回 `POST /auth/login/wechat` 并在 `user` 表补 `wechat_open_id` 字段即可。
> - 老人账号可由监护人在小程序内代建，或由 Web 管理后台创建（若全部由后台代建，可去掉 `/auth/register`）。

**示例：账号密码登录**

```
POST /auth/login
{ "account":"13800001234", "password":"******" }

// 响应
{ "code":0, "data":{ "token":"eyJhbGciOi...", "refreshToken":"...",
                     "userId":"u-1001", "role":"GUARDIAN", "expiresIn":7200 } }
```

## 4. 设备管理（命令实际经 MQTT 下行）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /devices/register | 【鉴权】设备注册绑定到老人（准入审核在 Web 后台完成，小程序端仅提交申请 / 查看状态） |
| GET | /devices | 【鉴权】我监护的老人设备列表（含在线状态） |
| GET | /devices/{deviceId} | 【鉴权】设备详情与最新遥测 |
| GET | /devices/{deviceId}/status | 【鉴权】实时状态（在线/仓门） |
| POST | /devices/{deviceId}/commands | 【鉴权】下发控制命令（蜂鸣/校准/解锁/重启），转 MQTT QoS1 |
| GET | /devices/{deviceId}/events | 【鉴权】设备事件流水（上下线、心跳异常） |

**示例：下发控制命令**

```
POST /devices/BOXA1001/commands
{ "cmd":"BUZZ", "params":{"durationSec":5}, "operator":"doctor" }

// 响应（异步下发，返回受理）
{ "code":0, "data":{ "commandId":"c-7f21", "state":"PENDING" } }
```

### 4.1 设备抓拍图片（摄像头留证，7 天过期）

摄像头识别结果的**结构化数据走 MQTT**（见文档 03 第 4 章）；**图片单独走 HTTP multipart 直传后端**。

> **当前摄像头默认全量抓拍：正常服药也拍、每次服药都拍**，后端按全量接收。摄像头侧的算法与抓拍策略不由后端 / 小程序团队负责，后端只负责"收下并管好"。

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /devices/{deviceId}/captures | 【**设备鉴权**】摄像头上传抓拍图片（multipart），返回 `captureId` 与 `expireAt` |
| GET | /captures?elderId=&reason=&from=&to= | 【鉴权】抓拍列表（仅元数据，不含文件流） |
| GET | /captures/{captureId} | 【鉴权】抓拍详情 |
| GET | /captures/{captureId}/file?token= | 【鉴权】读取图片流（供小程序 `<image src>` 使用，校验监护关系） |
| DELETE | /captures/{captureId} | 【鉴权·监护人】手动提前删除 |

**示例：摄像头上传抓拍**

```
POST /devices/BOXA1001/captures
Content-Type: multipart/form-data
file: capture.jpg
reason=WRONG_DRUG & medicineId=m-205 & planId=p-3301
& confidence=0.62 & capturedAt=2026-10-04T00:04:00Z

// 响应
{ "code":0, "data":{ "captureId":"cap-8801","expireAt":"2026-10-11T00:04:00Z" } }
```

**约定**：

- **`reason` 取值**：`NORMAL` 正常服药抓拍（**当前占绝大多数**，摄像头全量拍）、`WRONG_DRUG` 错服、`LOW_CONFIDENCE` 置信度低于阈值、`MANUAL` 监护人主动请求。列表接口可按 `reason=WRONG_DRUG` 过滤只看异常。
- **去重建议**：同一服药事件只保留 1 张（摄像头侧连拍时后端可按 `planId + capturedAt` 时间窗去重，或由摄像头侧控制）。
- **上传侧鉴权走设备身份，不走用户 JWT**：用注册时下发的 `deviceSecret` 做 HMAC 签名，Header 携带 `X-Device-Id` / `X-Device-Sign` / `X-Timestamp`，后端校验签名与时间窗；上传接口只允许该设备绑定老人下的写入。
- **下载侧鉴权**：必须是**老人本人**或其 **ACTIVE 监护人**，其余角色（护理 / 医生）一律 40301 —— 比 1.3 的默认规则更严格。
- **为什么用 query 传 token**：小程序 `<image src>` 无法携带自定义 Header，故用后端签发的**短期 `fileToken`**（5 分钟有效）拼接在 URL 上；**不要直接把长期 JWT 放 URL**（会落进日志与浏览器历史）。
- **过期清理（已读 / 未读分开算）**：`expire_at = MIN(captured_at + 30 天, viewed_at + 7 天)` —— **未读最长保留 30 天**，**首次查看后再保留 7 天**（查看只会让过期提前，不会延长）。首次调用 `/captures/{id}/file` 读取图片流时后端自动写入 `viewed_at` 并重算 `expire_at`；天数可配 `medbox.capture.retention-days-unread`（30）/ `retention-days-read`（7）。后端每日定时任务删除到期文件与记录，监护人可手动提前删除。
- **存储**：图片存后端主机本地目录，不落库、不暴露真实路径（表结构见文档 06 的 2.12）。

## 5. 药品与库存管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /medicines | 【鉴权】新建/更新药品档案（含禁忌、储存条件）；不传 `ownerElderId` 即公共字典 |
| GET | /medicines | 【鉴权】药品字典 / 我的药品列表（分页、搜索）；可见范围 = **公共药品（`ownerElderId` 为空）+ 我监护老人的私有药品** |
| GET | /medicines/{medicineId} | 【鉴权】药品详情 |
| GET | /devices/{deviceId}/sensors | 【鉴权】该药箱下的传感器子设备列表（温湿度 / 光照 / 摄像头，含各自在线状态） |
| GET | /devices/{deviceId}/stocks | 【鉴权】药箱库存清单（按药品：数量、生产日期、有效期） |
| PUT | /devices/{deviceId}/stocks/{medicineId} | 【鉴权】录入 / 更新某药品库存：数量、**生产日期与有效期（手动录入）** |
| POST | /devices/{deviceId}/stocks/{medicineId}/in | 【鉴权】补药入库（增加库存，可同时更新生产日期 / 有效期） |
| DELETE | /devices/{deviceId}/stocks/{medicineId} | 【鉴权】移除该药品库存 |
| GET | /devices/{deviceId}/expiring | 【鉴权】临期/过期清单（提前 N 天预警） |

> **药箱不划分仓位 / 格子**：库存按"药箱 + 药品"管理（表 `medicine_stock`，见文档 06 的 2.5）；传感器则按**子设备**细分（`device.device_type` + `parent_device_id`），见文档 06 的 2.3。

## 6. 服药计划与提醒

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /plans | 【鉴权】创建服药计划（**一次提醒可含多种药品**，请求体带 `items` 数组） |
| PUT | /plans/{planId} | 【鉴权】修改计划（整体覆盖：时间/周期/报警规则与 `items` 明细一并提交） |
| PATCH | /plans/{planId}/status | 【鉴权】启用/停用计划 |
| DELETE | /plans/{planId} | 【鉴权】删除计划（级联删除明细） |
| GET | /plans?elderId= | 【鉴权】按老人查询计划列表（列表项含药品概览） |
| GET | /plans/{planId} | 【鉴权】计划详情（含 `items` 明细） |
| GET | /plans/{planId}/next-doses | 【鉴权】未来待服药时间线（按次展开，每次列出所需药品） |

计划保存后，后端将提醒规则转成设备可执行的定时任务，并通过 MQTT 下行同步给设备；提醒触发以设备本地为准，云端做兜底与统计。

> **一次提醒 = 一个计划，可含多种药**：`times` / `repeatRule` / 报警规则在计划头统一配置，药品与剂量放在 `items` 明细里（**无仓位字段**，见文档 06 的 2.6 / 2.6.1）。漏服判定以"本次所有 item 都有服药记录"为准，缺任一种药即判 MISS。
>
> **两处口径**：① `times` 为**设备本地墙钟时间**（时区取 `device.timezone`，默认 `Asia/Shanghai`），不是 UTC；② 漏服容忍时长优先级为 **`missAlarmAfterMin`（计划级）> `alarm_setting.missTolerateMin`（老人级）> 系统默认 15 分钟**，计划级留空即沿用上一级。

**示例：创建服药计划（两种药同时服用）**

```json
POST /plans
{
  "elderId":"e-1001","name":"早餐后",
  "times":["08:00","20:00"],"repeatRule":"DAILY","missAlarmAfterMin":15,
  "alarmRule":{"miss":true,"wrongDrug":true,"expired":true,"env":true},
  "items":[
    { "medicineId":"m-205","dose":"1","unit":"片" },
    { "medicineId":"m-388","dose":"2","unit":"粒","note":"餐后" }
  ]
}
// 响应
{ "code":0, "data":{ "planId":"p-3301","syncState":"SYNCED_TO_DEVICE",
    "items":[ { "itemId":"pi-9001","medicineId":"m-205" },
              { "itemId":"pi-9002","medicineId":"m-388" } ] } }
```

## 7. 服药记录与依从性

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /records?elderId=&from=&to= | 【鉴权】服药记录分页（含是否按时/是否错服） |
| GET | /records/{recordId} | 【鉴权】记录详情（关联计划与抓拍留证） |
| GET | /adherence?elderId=&period=day\|week\|month | 【鉴权】依从性统计（按时率/漏服/错服） |
| POST | /records/{recordId}/confirm | 【鉴权】监护人补录/确认（设备异常时手工校准） |
| GET | /records/export?elderId=&from=&to= | 【鉴权】导出服药报告（护理/医生用） |

## 8. 告警（列表/处理，实时推送走 WebSocket）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /alarms?elderId=&type=&readState= | 【鉴权】告警分页（漏服/错服/过期/环境超标） |
| GET | /alarms/unread-count | 【鉴权】未读告警数（小程序角标） |
| PATCH | /alarms/{alarmId}/read | 【鉴权】标记已读 |
| PATCH | /alarms/{alarmId}/handle | 【鉴权】处理/确认告警（护理人员闭环） |
| PUT | /alarm-settings?elderId= | 【鉴权】配置告警规则与阈值（提前期/环境上下限/漏服容忍） |
| GET | /alarm-settings?elderId= | 【鉴权】查询当前告警配置 |

**告警类型枚举**

| type | 触发来源 | 说明 |
|------|----------|------|
| MISS | 后端判定 | 计划时间+容忍时长内未收到该计划服药事件 |
| WRONG_DRUG | 设备上报 | 摄像头识别的实际药品 ≠ 计划药品 |
| EXPIRED | 后端判定 | 按手动录入的 `expiryDate` 扫描：临期预警 + 到期告警 |
| ENV | 设备上报 | 温度/湿度/光照超出该药品储存阈值 |
| DEVICE_OFFLINE | 后端判定 | 心跳超时，设备离线超阈值 |

## 9. 环境监测

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /env/realtime?deviceId= | 【鉴权】最新温湿光读数 |
| GET | /env/history?deviceId=&from=&to=&interval= | 【鉴权】历史曲线（小程序图表） |

> **环境阈值按老人统一配置**（不再按药品单独设置），走 `PUT /alarm-settings?elderId=` 的 `envThreshold` 字段（见第 8 章）；判定口径见文档 01 的 5.4。
