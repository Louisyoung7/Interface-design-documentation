# 变更记录（CHANGELOG）

> 本文件记录文档集的每次版本变更，便于三个代码仓库（**Java 后端 / uni-app 小程序 / Web 管理后台**）判断"要不要同步、影响哪些模块"。
> 同步方式见 `00_README_文档索引.md` 的《文档如何同步到代码仓库》。
> 版本格式：文档集统一版本号（各文件头部标注），打 tag 后代码仓库可按 tag 固定拉取。

## [V1.5] — 2026-10-05

### 01 系统架构与部署
- 新增**双人独立开发**方案：IP 只写在 `.env.local` / `application-local.yml`（不入库），`git pull` 无需改动任何配置
- 新增 `application.yml` 说明与"为什么监听 `0.0.0.0` 不能省掉地址配置"的辨析
- 新增**角色权限默认规则**（角色 × 权限矩阵 + 四步校验顺序）
- 明确**时区口径**：服药计划 `times` 为设备本地墙钟时间（`device.timezone`），其余一律 UTC
- 明确**漏服容忍优先级**：计划级 > 老人级 > 系统默认 15 分钟
- **药箱不划分仓位**；设备拆分为药箱主控 + 传感器子设备
- 新增 4.6 抓拍图片的存储、鉴权与过期规则（已读 7 天 / 未读 30 天）
- 附录新增《凭据的"加密存储"——防什么、什么时候做》
- **删除 OTA / 固件版本**相关内容（仅保留一句范围说明）
- 5.2 错服识别改为摄像头识别药品；5.3 过期改为按 `medicine_stock.expiry_date`

### 02 REST 接口设计
- 登录改为**账号 + 密码**，删除短信验证码登录与微信登录（`wechat_open_id` 预留）
- 新增 1.3 权限默认规则与 `40302`（监护关系未生效）
- 监护绑定改为**两步**：监护人发起（PENDING）→ 老人端确认（ACTIVE）
- 服药计划支持**一次提醒多种药**（`items` 数组，无仓位字段）
- **仓位接口删除**，改为库存接口 `/devices/{id}/stocks*`；`/devices/{id}/sensors` 归入设备管理
- 新增 `DELETE /medicines/{medicineId}`；删除按药品设置的环境阈值接口
- 新增 4.1 设备抓拍图片接口（上传 / 列表 / 详情 / 读取文件流 / 删除）
- 环境阈值统一为老人级；告警枚举 `ENV`、`WRONG_DRUG` 描述同步更新
- 错误码表排序整理；设备注册改为"注册即生效，不做准入审核"

### 03 设备接入协议 MQTT
- 新增 `up/ack` 回执主题与超时重发；上行统一带 `msgId` 做幂等去重
- Topic 前缀去掉前导 `/`；明确**一个药箱只有主控建 MQTT 连接**
- 新增第 4 章**摄像头识别链路**（WiFi → 单片机 → MQTT；图片 HTTP 直传不过 MQTT）
- 遥测改为**每个传感器各发一条**；新增 `up/sensor/heartbeat` 判子设备离线
- 明确**错服只走 `dispense.wrongDrug`**，`up/event/error` 仅用于设备侧异常
- 低置信度**不判错服**，抓拍标 `LOW_CONFIDENCE` 交人工复核
- 补充 `up/inventory` payload 示例；`dispense` 增加 `confidence` / `source` / `imageId`
- **删除一机一密章节**；明确本期为匿名连接，两级加固仅在 03 / 01 附录各提一句

### 04 实时推送协议 WebSocket
- 告警 `level` 统一为 `INFO / WARN / CRITICAL`（示例原为 `HIGH`）
- `REMINDER` 改为 `medicines[]`（一次提醒含多种药）
- `DEVICE_STATUS` 增加 `deviceType` / `parentDeviceId`，覆盖传感器子设备；去掉 `battery` 与 `firmwareVer`

### 05 AI 大模型与 RAG 方案
- **OCR 与 Embedding 本地化**：RapidOCR 侧车服务 + Ollama `bge-m3`（1024 维），不出网、无需 Key
- 向量维度 1536 → **1024**（换模型须重建全部分块向量）
- 新增 5.1 RAG 检索路径（锁定老人 → 定位 medicineId → 带过滤召回 → 兜底说明）
- 补全会话接口（创建 / 重命名 / 删除，首次提问隐式创建 `sessionId`）

### 06 数据库设计
- **删除 `compartment`**，新增 `medicine_stock`（药箱 + 药品维度）
- `device` 增加 `device_type` 与 `parent_device_id`（主控 / 温湿度 / 光照 / 摄像头）
- 新增 `capture` 抓拍表（含 `viewed_at`、`expire_at`）
- `med_plan_item`、`med_record` 删除 `slot_no`；`med_record` 增加 `capture_id`
- `medicine` 增加 `owner_elder_id`（公共字典 / 私有药品）
- `med_record`、`env_sample` 冗余 `elder_id`；`env_sample` 增加 `gateway_device_id`
- `user` 增加 `username`、`password_hash`，`phone` 改 UNIQUE
- `drug_manual_chunk` 外键统一为 `VARCHAR(64)`
- `device.secret` 标注为预留未使用（本期匿名，无设备凭据）

### 影响提示

| 代码仓库 | 是否需要改代码 | 说明 |
|----------|----------------|------|
| Java 后端 | ✅ 需要 | 计划主子表、库存表、抓拍表与接口、本地 OCR/Embedding、匿名 MQTT 连接 |
| uni-app 小程序 | ✅ 需要 | 登录改账号密码、计划改为 `items`、监护两步确认、抓拍图片查看 |
| 嵌入式（设备端） | ✅ 需要 | 上行统一带 `msgId`、`up/ack` 回执、遥测按传感器分条、传感器心跳、摄像头链路 |
| Web 管理后台 | ➖ 后续 | 本期文档基本未涉及，待设计 |

## [V1.4] — 2026-10-03（基线）

文档集初始版本：系统架构与部署（01）、REST 接口设计（02）、设备接入协议 MQTT（03）、实时推送协议 WebSocket（04）、AI 大模型与 RAG 方案（05）、数据库设计（06）。
