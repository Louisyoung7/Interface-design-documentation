# 05 · AI 大模型接入与 RAG 方案（Spring Boot + Spring AI）

> 后端：Spring Boot　|　数据库：PostgreSQL（pgvector，建表见文档 06）　|　**OCR 与 Embedding：本地部署（不出网、无需 Key）**　|　**Chat 问答：OpenAI 兼容 API，API Key 由监护人代配**　|　版本 V1.5

## 1. 接入方式：Java 后端直连大模型

后端（Spring Boot）直接调用大模型 OpenAI 兼容的 `/chat/completions` 接口，不引入独立的 Python 中转服务。

1. 同步问答：普通 HTTP POST，超时（如 15s）后降级返回模板化安全提示（错误码 50310）；
2. 流式问答：用支持 SSE 的客户端消费上游 `text/event-stream`，边收边向 `/ai/chat/stream` 转发 `delta`；
3. 轻量 RAG：知识库检索在 Java 内完成，召回片段拼进 Prompt（见第 5 章）；
4. 统一在响应处注入 `disclaimer`（仅供参考、遵医嘱），对涉医疗结论做后置过滤。

RAG 框架采用 **Spring AI + `PgVectorStore`**：与 Spring Boot 原生装配，自带 `ChatClient`、`EmbeddingModel`、`VectorStore` 与 RAG Advisor，OpenAI 兼容接口开箱可用；向量表结构与索引定义见文档 06，不在此重复。

**出网范围**：只有 **Chat 问答**需要出网（放行监护人所选模型的域名），鉴权与 Key 管理内聚在后端。**OCR 与 Embedding 均在本地完成，不出网、不需要 Key**。

## 2. API Key 由监护人代老人配置

- 一个老人一份配置（`LlmConfig`），仅监护人可写、可测、可删；老人端只问答、不接触 Key。
- Key 在服务端加密存储（AES，密钥与库分离），日志与回显全程脱敏。**为什么它必须加密存储，见文档 01 附录《凭据的"加密存储"》**。
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
| POST | /ai/chat | 【鉴权】同步问答（question + 可选 elderId/deviceId/sessionId 上下文） |
| POST | /ai/chat/stream | 【鉴权】流式问答（SSE），长回答逐字输出 |
| POST | /ai/sessions | 【鉴权】创建会话，返回 `sessionId` |
| GET | /ai/sessions?elderId= | 【鉴权】历史会话列表 |
| GET | /ai/sessions/{sessionId}/messages | 【鉴权】会话消息明细 |
| PATCH | /ai/sessions/{sessionId} | 【鉴权】重命名会话（可选） |
| DELETE | /ai/sessions/{sessionId} | 【鉴权】删除会话及其消息 |

> **会话规则**：`sessionId` 可省略 —— 首次调用 `/ai/chat`（或 `/ai/chat/stream`）时若未传 `sessionId`，后端**隐式创建**一个新会话并在响应中返回 `sessionId`，前端据此续接同一轮对话。会话归属 `elderId`，切换老人即切换会话列表。

> 老人端不做 API Key 相关操作；老人发起提问时，后端按 elderId 取出监护人配好的 Key 调用大模型。

**示例：AI 问答**

```json
POST /ai/chat
{ "question":"阿莫西林还有半年过期能吃吗？和头孢冲突吗？",
  "elderId":"e-1001", "scene":"MEDICINE_QUERY" }

{ "code":0, "data":{
    "sessionId":"s-7001",
    "answer":"库存显示该阿莫西林有效期至 2027-04，未过期可正常服用；头孢与阿莫西林同属β-内酰胺类，过敏史需注意……（仅供参考，遵医嘱）",
    "citations":["库存#m-205 有效期2027-04","知识库#beta-lactam"],
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

**本期录入实现范围**：上传图片/PDF → **OCR 转文本（默认本地 OCR，云端 API 为可切换备选）** → 切块 → Embedding → 写入 pgvector；召回时按 medicineId / elderId 过滤、答案带出处。不做版面分析（表格 / 多页 / 图示还原）。

### 4.1 OCR 方案：本地优先，云端备选

| 方案 | 费用 | 中文印刷体效果 | 部署成本 | 说明 |
|------|------|----------------|----------|------|
| **RapidOCR（ONNX，纯 CPU）** ← 默认 | 免费 | 好（接近 PaddleOCR） | 小：`pip install rapidocr-onnxruntime` + FastAPI 包一层，模型约 20MB | 无重依赖、启动快，局域网内闭环 |
| PaddleOCR | 免费 | 最好 | 中：PaddlePaddle 依赖较大，CPU 推理偏慢 | 效果优先时选它 |
| Tesseract + `chi_sim`（tess4j） | 免费 | 一般（说明书小字 / 复杂排版易掉字） | 最小：Java JNI 直接调，无需独立服务 | 仅当"不想起额外服务"时考虑 |
| 云端 OCR API（腾讯 / 百度 / 阿里） | 有免费额度，超出按次计费（约几分~几毛 / 次） | 好 | 需注册 + **个人实名认证**、配 Key、后端放行出网域名 | 见下方说明 |
| 多模态大模型直接读图 | 按 token 计费 | 好（取决于模型） | **零新增部署**（复用已有大模型通道） | 见 4.2 |

**云端 OCR API 的实际情况**：

- **要不要钱**：主流厂商均提供**每月免费额度**（通常数百~数千次，以官网为准）；说明书录入是低频操作（一药一次），演示阶段基本用不到付费额度。
- **要不要认证**：需要**实名认证**，个人实名即可，不要求企业主体、不要求微信认证。
- **代价**：多一个 Key 要管理、后端需放行出网、且**说明书含用药信息，健康数据会出网**。

**结论：本项目默认走本地 OCR** —— 零费用、无需实名、数据不出内网，与"老人用药属敏感健康数据"的合规要求一致（见文档 01 附录第 5 条）。

- **部署形态**：一个 Python 侧车服务 `ocr-service`（FastAPI），`POST /ocr` 收图片 / PDF，返回按行 / 段落组织的纯文本；Java 侧抽象为 `OcrClient` 接口，实现类可切换：`medbox.ocr.provider=local | cloud`（`LocalOcrClient` / `CloudOcrClient`）。
- **PDF**：带文字层的 PDF 直接用 `pdfbox` 抽取文本，**跳过 OCR**（更快更准）；扫描件则渲染成页图（`pypdfium2` / `pdf2image`）再逐页 OCR。
- **提效**：OCR 前做简单预处理（灰度、去噪、可选透视矫正），能明显提升说明书小字的识别率。
- **Embedding 同样本地化**（Ollama + `bge-m3`，1024 维，无 Key、不出网，见 4.3）。这样整条**录入链路（OCR → 切块 → 向量化 → 入库）完全在局域网内闭环**，出网只剩 Chat 问答一项。

### 4.2 备选：多模态大模型直接读图

原"不做多模态直接读图"的限制放宽为：**默认不用，作为 OCR 失败时的兜底重试**。若监护人为老人配置的模型支持视觉，可把说明书图片直接交给它转写为文本，不新增任何部署。成本按 token 计，且只发生在**录入**环节（低频），不在问答环节。

### 4.3 Embedding 本地化：Ollama + bge-m3

录入链路（OCR → 切块 → 向量化 → 入库）**完全在局域网内闭环**，不需要任何 Key。

- **部署**：本机起 Ollama（`http://localhost:11434`），`ollama pull bge-m3`。Spring AI 原生提供 Ollama 的 `EmbeddingModel`，直接装配即可，无需自写客户端。
- **配置**：

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      embedding:
        model: bge-m3
    vectorstore:
      pgvector:
        dimensions: 1024              # 必须与 bge-m3 一致
        distance-type: COSINE_DISTANCE
        index-type: HNSW
```

- **维度**：`bge-m3` = **1024 维**，文档 06 的 `embedding vector(1024)` 已同步。
- **⚠️ 换模型必须重建向量**：维度一旦确定就不能只改配置。更换 embedding 模型或维度后，需**清空 `drug_manual_chunk` 并对全部说明书重新解析**（批量走 `/medicines/{id}/manuals/{manualId}/reprocess`），否则新旧向量混在同一索引里，检索结果无意义。
- **为什么 embedding 不做"每老人一份"**：向量空间必须全局一致，才能跨药品 / 跨老人检索与比较，因此 embedding 与老人无关 —— 本地部署后自然也不存在 Key 的问题。
- **资源占用**：`bge-m3` 约 1.2GB，CPU 可跑（数十~数百 ms / 段）；入库是异步任务，不阻塞问答；有 GPU 时 Ollama 会自动利用。

**为什么不直接用 llama.cpp**：Ollama 的底层就是 llama.cpp，等于"llama.cpp + 模型仓库 + 常驻服务 + OpenAI 兼容 API + 自动 GPU/CPU 调度"的省心版，能力上并没有损失。直接上 llama.cpp 需要自己找 GGUF 量化版、手动指定 `--embedding` 与 `--pooling`（配错则向量质量直接崩）、自己起 `llama-server` 并管进程与开机自启，且 Spring AI 没有专用 starter（只能把 OpenAI 兼容 starter 指向本地并伪造 api-key）。除非将来要把 **Chat 也本地化**并精细控制量化 / 并发 / 显存（那种场景更适合 llama.cpp 或 vLLM），当前只做 embedding 时 Ollama 更省事。

**更轻的备选**：若嫌 1.2GB 太大，可让 Python 侧车服务同时承担 OCR 与 Embedding，用 ONNX 直接加载 `bge-small-zh-v1.5`（约 100MB，**512 维**，中文场景效果不错且更快）。代价是该服务需自行暴露 `/v1/embeddings` 供 Spring AI 的 OpenAI 兼容 starter 调用，且文档 06 的维度要改为 512 并重建全部分块向量。

入库与问答链路：

```
监护人上传说明书(图片/PDF) → Java 后端接收
  → 调 OCR 服务转纯文本（本地 RapidOCR 侧车服务，或云端 API，见 4.1）
  → 按 500~1000 字 / 段落切块
  → 调本地 Embedding 服务生成向量（Ollama：bge-m3，1024 维，见 4.3）
  → 写入 pgvector（drug_manual_chunk）
问答时：问题 → Embedding → pgvector Top-K 检索（带 medicineId 过滤）
  → 片段拼进 Prompt → Chat API 生成 → SSE 流式返回小程序
```

## 5. 两类知识分开处理

| 知识类型 | 例子 | 存储 | 获取方式 |
|----------|------|------|----------|
| 结构化 | 库存数量、有效期、剂量、服药计划、依从性 | PostgreSQL 关系表 | 直接 SQL 查询，不交给大模型判断 |
| 非结构化 | 说明书正文、禁忌、不良反应、相互作用 | PostgreSQL + pgvector 向量列 | RAG 语义检索 Top-K 片段 |

问答时后端把两者拼进同一个 Prompt：**库存有效期 + 禁忌（结构化）+ 召回的说明书片段（向量）+ 用户问题**，再交给大模型生成，最后注入 `disclaimer`。"是否过期"以库存表有效期字段为准，说明书片段只用于解释"禁忌 / 相互作用"。

### 5.1 RAG 检索路径（问题 → 定位药品 → 带过滤召回）

**不能一上来就全库向量检索** —— 那样会召回别人的或其他药品的片段。固定四步：

1. **锁定老人范围**：用请求中的 `elderId` 查出该老人**当前在库 / 在服的药品集合**（`medicine_stock` + `med_plan_item`），这是召回的硬边界。
2. **定位目标药品 `medicineId`**（依次尝试，命中即停）：
   - 请求显式传了 `medicineId` → 直接用；
   - 否则用**在库药品名做字符串包含 / 模糊匹配**（"阿莫西林还有半年过期能吃吗" → 命中"阿莫西林胶囊"）；
   - 仍匹配不到 → **不猜**，把范围放宽为"该老人全部在库药品"，避免答非所问。
3. **带过滤的向量召回**（Top-K 建议 4~6，余弦距离）：

```sql
SELECT chunk_id, medicine_id, content, page,
       embedding <=> :qvec AS distance
FROM drug_manual_chunk
WHERE medicine_id = ANY(:medicineIds)              -- 第 2 步定位到的药品
  AND (elder_id = :elderId OR elder_id IS NULL)    -- 私有 + 公共说明书
ORDER BY distance
LIMIT 5;
```

4. **拼装与兜底**：结构化事实（库存 / 有效期 / 禁忌）+ 召回片段（带 `chunk_id` 便于溯源）+ 问题 → Chat。**若召回为空或最近距离仍低于相似度阈值**，只用结构化数据作答，并在 `disclaimer` 中说明"未在说明书中找到依据"，避免模型凭空编造。

> 只有**文档的写入（录入）环节**才需要拿全部说明书；问答环节永远先收敛 `medicineIds` 再检索。

## 6. 与"每老人一份 Key"的结合（关键实现点）

Spring AI 自动装配的 `ChatModel` 是单例、读全局 `api-key`；本项目 Key 按 elderId 存于 `llm_config`，因此：

- **Embedding 模型**：**本地 Ollama + `bge-m3`（1024 维）**，全局唯一、与老人无关、**不需要 Key、不出网**（见 4.3）。它是全局单例 bean 即可；换模型须重建全部分块向量；
- **Chat 问答**：在每次请求内按 elderId 取出该老人的 `{baseUrl, apiKey, model}`，动态构造 ChatClient/ChatModel（或用 OpenAI 兼容客户端覆盖请求头 `Authorization`），实现"每个老人用监护人配的 Key 调用"，不要直接注入全局单例 bean；
- 未配置 / Key 无效：走友好提示 + 50310，不回显明文 Key。

> 流式问答把同步调用换成流式（`Flux<String>`），在 `/ai/chat/stream` 以 SSE 转发 `delta`；若走 WebSocket 备选通道，则按文档 04 的 `AI_STREAM` 帧推送。
