# 05 · AI 大模型接入与 RAG 方案（Spring Boot + Spring AI）

> 后端：Spring Boot　|　数据库：PostgreSQL（pgvector，建表见文档 06）　|　大模型：OpenAI 兼容 API，API Key 由监护人代配　|　版本 V1.4

## 1. 接入方式：Java 后端直连大模型

后端（Spring Boot）直接调用大模型 OpenAI 兼容的 `/chat/completions` 接口，不引入独立的 Python 中转服务。

1. 同步问答：普通 HTTP POST，超时（如 15s）后降级返回模板化安全提示（错误码 50310）；
2. 流式问答：用支持 SSE 的客户端消费上游 `text/event-stream`，边收边向 `/ai/chat/stream` 转发 `delta`；
3. 轻量 RAG：知识库检索在 Java 内完成，召回片段拼进 Prompt（见第 5 章）；
4. 统一在响应处注入 `disclaimer`（仅供参考、遵医嘱），对涉医疗结论做后置过滤。

RAG 框架采用 **Spring AI + `PgVectorStore`**：与 Spring Boot 原生装配，自带 `ChatClient`、`EmbeddingModel`、`VectorStore` 与 RAG Advisor，OpenAI 兼容接口开箱可用；向量表结构与索引定义见文档 06，不在此重复。

出网仅需为后端主机放行到模型域名的访问，鉴权与 Key 管理内聚在后端。

## 2. API Key 由监护人代老人配置

- 一个老人一份配置（`LlmConfig`），仅监护人可写、可测、可删；老人端只问答、不接触 Key。
- Key 在服务端加密存储（如 AES，密钥与库分离），日志与回显全程脱敏。
- 未配置 / Key 无效时，老人端问答返回友好提示"AI 助手暂未开通，请联系您的监护人配置"（50310）。
- 对每个 Key 设每日调用次数 / Token 上限（配合 42901），防止误刷产生费用。

### 2.1 大模型接入配置接口（监护人代配）

| 方法 | 路径 | 说明 |
|------|------|------|
| PUT | /ai/config?elderId= | 【鉴权·仅监护人】为指定老人保存配置（provider、apiKey、model、baseUrl） |
| GET | /ai/config?elderId= | 【鉴权】查询配置（apiKey 脱敏回显，如 `sk****abcd`） |
| POST | /ai/config/test?elderId= | 【鉴权·仅监护人】用当前 Key 发一次探测请求，验证连通/额度 |
| DELETE | /ai/config?elderId= | 【鉴权·仅监护人】清除该老人的配置 |

**示例：监护人保存某老人的大模型配置**

```
PUT /ai/config?elderId=e-1001
{ "provider":"openai-compatible",
  "apiKey":"sk-xxxxxxxxxxxxxxxx",
  "model":"gpt-4o-mini",
  "baseUrl":"https://api.openai.com/v1",
  "enabled":true }

// 响应（不回显明文 Key）
{ "code":0, "data":{ "elderId":"e-1001","provider":"openai-compatible",
  "model":"gpt-4o-mini","apiKeyMask":"sk****xxxx","enabled":true } }
```

## 3. AI 药品问答接口

小程序以会话方式向 AI 助手提问（是否过期、服用禁忌、相互作用等）。后端做鉴权、上下文拼装与 RAG 检索，直接转调大模型；所用 API Key 为该老人由监护人预先配置的那一份（见 2.1）。

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /ai/chat | 【鉴权】同步问答（question + 可选 elderId/deviceId 上下文） |
| POST | /ai/chat/stream | 【鉴权】流式问答（SSE），长回答逐字输出 |
| GET | /ai/sessions?elderId= | 【鉴权】历史会话列表 |
| GET | /ai/sessions/{sessionId}/messages | 【鉴权】会话消息明细 |

> 老人端不做 API Key 相关操作；老人发起提问时，后端按 elderId 取出监护人配好的 Key 调用大模型。

**示例：AI 问答**

```json
POST /ai/chat
{ "question":"阿莫西林还有半年过期能吃吗？和头孢冲突吗？",
  "elderId":"e-1001", "scene":"MEDICINE_QUERY" }

{ "code":0, "data":{
    "answer":"库存显示该阿莫西林有效期至 2027-04，未过期可正常服用；头孢与阿莫西林同属β-内酰胺类，过敏史需注意……（仅供参考，遵医嘱）",
    "citations":["库存#slot3 有效期2027-04","知识库#beta-lactam"],
    "disclaimer":"本建议仅供参考，用药请遵医嘱。" } }
```

## 4. 药品说明书知识库（拍照/上传录入）

药品说明书等非结构化内容由**监护人**拍照或上传，后端经 OCR 转文本、切块、向量化后写入 pgvector（表结构见文档 06）。老人端只问答、不录入。

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /medicines/{medicineId}/manuals | 【鉴权·监护人】上传说明书图片/PDF（multipart，异步触发 OCR→切块→embedding→入库） |
| GET | /medicines/{medicineId}/manuals | 【鉴权】查询该药品已录入的说明书列表（含解析状态） |
| POST | /medicines/{medicineId}/manuals/{manualId}/reprocess | 【鉴权·监护人】OCR 失败或换版本后重新解析入库 |
| DELETE | /medicines/{medicineId}/manuals/{manualId} | 【鉴权·监护人】删除说明书及其向量分块 |
| GET | /medicines/{medicineId}/manuals/{manualId}/chunks | 【鉴权】查看切块结果（核对 OCR 质量，分页） |

> 上传接口直接返回受理态（`status: PARSING`），OCR + 向量化为后台异步任务，完成置为 `INDEXED`；小程序用 GET 列表轮询或经 WebSocket 通知刷新。

**示例：上传说明书（multipart）**

```
POST /medicines/m-205/manuals
Content-Type: multipart/form-data
file: 阿莫西林说明书.jpg        // 或 .pdf

// 响应（异步受理，不回显向量）
{ "code":0, "data":{ "manualId":"dm-77", "status":"PARSING" } }
```

**本期录入实现范围**：上传图片/PDF → 第三方 OCR API 转文本 → 切块 → Embedding → 写入 pgvector；召回时按 medicineId / elderId 过滤、答案带出处。不做版面分析（表格 / 多页 / 图示还原）与多模态直接读图。

入库与问答链路：

```
监护人上传说明书(图片/PDF) → Java 后端接收
  → 调第三方 OCR API 转纯文本
  → 按 500~1000 字 / 段落切块
  → 调 Embedding API 生成向量
  → 写入 pgvector（drug_manual_chunk）
问答时：问题 → Embedding → pgvector Top-K 检索（带 medicineId 过滤）
  → 片段拼进 Prompt → Chat API 生成 → SSE 流式返回小程序
```

## 5. 两类知识分开处理

| 知识类型 | 例子 | 存储 | 获取方式 |
|----------|------|------|----------|
| 结构化 | 库存数量、有效期、仓位、剂量、服药计划、依从性 | PostgreSQL 关系表 | 直接 SQL 查询，不交给大模型判断 |
| 非结构化 | 说明书正文、禁忌、不良反应、相互作用 | PostgreSQL + pgvector 向量列 | RAG 语义检索 Top-K 片段 |

问答时后端把两者拼进同一个 Prompt：**库存有效期 + 禁忌（结构化）+ 召回的说明书片段（向量）+ 用户问题**，再交给大模型生成，最后注入 `disclaimer`。"是否过期"以库存表有效期字段为准，说明书片段只用于解释"禁忌 / 相互作用"。

## 6. 与"每老人一份 Key"的结合（关键实现点）

Spring AI 自动装配的 `ChatModel` 是单例、读全局 `api-key`；本项目 Key 按 elderId 存于 `llm_config`，因此：

- **Embedding 模型**：用一把固定的系统级 Key（说明书入库统一 embedding 模型与维度，保证向量空间一致），与老人无关；
- **Chat 问答**：在每次请求内按 elderId 取出该老人的 `{baseUrl, apiKey, model}`，动态构造 ChatClient/ChatModel（或用 OpenAI 兼容客户端覆盖请求头 `Authorization`），实现"每个老人用监护人配的 Key 调用"，不要直接注入全局单例 bean；
- 未配置 / Key 无效：走友好提示 + 50310，不回显明文 Key。

> 流式问答把同步调用换成流式（`Flux<String>`），在 `/ai/chat/stream` 以 SSE 转发 `delta`；若走 WebSocket 备选通道，则按文档 04 的 `AI_STREAM` 帧推送。
