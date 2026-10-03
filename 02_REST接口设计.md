# 02 · REST 接口设计

> 所有路径均相对 Base URL：`http://{后端主机内网IP}:8080/medbox/api/v1`　|　版本 V1.4
> 标注【鉴权】表示需携带 JWT，监护数据类接口需校验权限。AI 相关接口见文档 05。

## 1. 通用约定

| 项 | 约定 |
|----|------|
| Base URL | `http://{后端主机内网IP}:8080/medbox/api/v1`（局域网内网地址，见文档 01） |
| 请求体格式 | `application/json; charset=utf-8` |
| 认证方式 | Header: `Authorization: Bearer {JWT}`（设备端走 MQTT 用户名密码 / 一机一密，不走此通道） |
| 时间格式 | ISO-8601 UTC，如 `2026-10-03T07:04:00Z`；前端本地化展示 |
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
| 40301 | 403 | 无权限（越权访问他人数据 / 设备） |
| 40401 | 404 | 资源不存在 |
| 40901 | 409 | 状态冲突（如设备离线无法下发命令） |
| 42901 | 429 | 触发限流 |
| 50000 | 500 | 服务端内部错误 |
| 50310 | 503 | AI 大模型服务暂不可用 / API Key 无效 |

## 2. 核心数据模型

> 全部实体清单与建表 DDL 见文档 06《数据库设计》。本文件只列接口用到的实体名，便于对照路径含义。

涉及实体：User（老人/监护人）、GuardianRelation、Device、Medicine、Compartment（仓位）、MedPlan（服药计划）、MedRecord（服药记录）、EnvSample（环境遥测）、Alarm（告警）、AlarmSetting（告警规则）、LlmConfig（大模型配置）、DrugManual（说明书）、DrugManualChunk（说明书向量分块，见文档 05 / 06）。


## 3. 认证与用户

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /auth/login/sms | 手机号+验证码登录/注册，返回 JWT |
| POST | /auth/login/wechat | 小程序 code 换取登录态（openid 绑定） |
| POST | /auth/refresh | 刷新 Token |
| GET | /users/me | 【鉴权】当前用户资料与角色 |
| POST | /users/{elderId}/guardians | 【鉴权】老人绑定监护关系（需老人端授权码） |
| DELETE | /users/guardians/{relationId} | 【鉴权】解除监护关系 |
| GET | /users/{elderId}/profile | 【鉴权】老人基础信息（监护人可见） |

## 4. 设备管理（命令实际经 MQTT 下行）

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /devices/register | 【鉴权】设备注册绑定到老人（准入审核在 Web 后台完成，小程序端仅提交申请 / 查看状态） |
| GET | /devices | 【鉴权】我监护的老人设备列表（含在线状态） |
| GET | /devices/{deviceId} | 【鉴权】设备详情与最新遥测 |
| GET | /devices/{deviceId}/status | 【鉴权】实时状态（在线/电量/仓门/固件版本） |
| POST | /devices/{deviceId}/commands | 【鉴权】下发控制命令（蜂鸣/校准/解锁/重启），转 MQTT QoS1 |
| GET | /devices/{deviceId}/events | 【鉴权】设备事件流水（上下线、心跳异常） |

> 固件 OTA 触发属管理员职责，归 Web 管理后台，不在小程序接口内。

**示例：下发控制命令**

```
POST /devices/BOXA1001/commands
{ "cmd":"BUZZ", "params":{"durationSec":5}, "operator":"doctor" }

// 响应（异步下发，返回受理）
{ "code":0, "data":{ "commandId":"c-7f21", "state":"PENDING" } }
```

## 5. 药品与库存管理

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /medicines | 【鉴权】新建/更新药品档案（含禁忌、储存条件） |
| GET | /medicines | 【鉴权】药品字典/我的药品列表（分页、搜索） |
| GET | /medicines/{medicineId} | 【鉴权】药品详情 |
| GET | /devices/{deviceId}/compartments | 【鉴权】设备各仓位库存与有效期 |
| PUT | /devices/{deviceId}/compartments/{slotNo} | 【鉴权】配置仓位：绑定药品、数量、有效期 |
| POST | /devices/{deviceId}/compartments/{slotNo}/in | 【鉴权】补药入库（增加库存） |
| DELETE | /devices/{deviceId}/compartments/{slotNo}/bind | 【鉴权】解绑仓位 |
| GET | /devices/{deviceId}/expiring | 【鉴权】临期/过期清单（提前 N 天预警） |

## 6. 服药计划与提醒

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /plans | 【鉴权】创建服药计划（药品/仓位/时间/剂量/周期/报警规则） |
| PUT | /plans/{planId} | 【鉴权】修改计划（远程调整提醒时间/剂量阈值） |
| PATCH | /plans/{planId}/status | 【鉴权】启用/停用计划 |
| DELETE | /plans/{planId} | 【鉴权】删除计划 |
| GET | /plans?elderId= | 【鉴权】按老人查询计划列表 |
| GET | /plans/{planId}/next-doses | 【鉴权】未来待服药时间线 |

计划保存后，后端将提醒规则转成设备可执行的定时任务，并通过 MQTT 下行同步给设备；提醒触发以设备本地为准，云端做兜底与统计。

**示例：创建服药计划**

```json
POST /plans
{
  "elderId":"e-1001","medicineId":"m-205","slotNo":3,
  "times":["08:00","20:00"],"dose":"1","unit":"片",
  "repeatRule":"DAILY","missAlarmAfterMin":15,
  "alarmRule":{"miss":true,"wrongDrug":true,"expired":true,"env":true}
}
// 响应
{ "code":0, "data":{ "planId":"p-3301","syncState":"SYNCED_TO_DEVICE" } }
```

## 7. 服药记录与依从性

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /records?elderId=&from=&to= | 【鉴权】服药记录分页（含是否按时/是否错服） |
| GET | /records/{recordId} | 【鉴权】记录详情（关联计划与仓位快照） |
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
| WRONG_DRUG | 设备上报 | RFID/识别模块判定取药与计划仓位不匹配 |
| EXPIRED | 设备/后端 | 仓位内药品到达有效期（提前预警+到期告警） |
| ENV | 设备上报 | 温度/湿度/光照超出该药品储存阈值 |
| DEVICE_OFFLINE | 后端判定 | 心跳超时，设备离线超阈值 |

## 9. 环境监测

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | /env/realtime?deviceId= | 【鉴权】最新温湿光读数 |
| GET | /env/history?deviceId=&from=&to=&interval= | 【鉴权】历史曲线（小程序图表） |
| PUT | /env/threshold?deviceId=&medicineId= | 【鉴权】按药品储存条件设置环境阈值 |
