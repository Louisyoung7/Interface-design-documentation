# 家用智能药品箱 · 后端 / 小程序接口设计文档（文档集）

> 范围：Java 后端（Spring Boot）+ uni-app 小程序　|　协议：HTTP REST、MQTT（EMQX）、WebSocket　|　版本 V1.6
> 通信方式：当前全部为局域网（内网）通信，无公网域名。
> 变更记录见 `CHANGELOG.md`；同步到代码仓库的方式见文末《文档如何同步到代码仓库》。

本文档集按主题拆分为多个文件，便于分工评审与联调。各文件内容互不重复，统一约定见 01，数据库设计统一见 06。

| 文件 | 内容 | 主要读者 |
|------|------|----------|
| `00_README_文档索引.md` | 本索引 | 全员 |
| `01_系统架构与部署.md` | 项目背景、协议分工、角色权限、局域网部署与地址规划（含双人独立开发的本地配置方案）、关键业务流程、安全加固附录 | 全员 |
| `02_REST接口设计.md` | REST 通用约定、统一响应与错误码、各业务模块接口（认证/设备/药品/计划/记录/告警/环境） | 后端 / 小程序 |
| `03_设备接入协议_MQTT.md` | EMQX 连接鉴权、Topic 规范、上下行 Payload 示例 | 后端 / 嵌入式 |
| `04_实时推送协议_WebSocket.md` | 连接鉴权、消息帧结构、推送事件类型 | 后端 / 小程序 |
| `05_AI大模型与RAG方案.md` | 大模型直连接入、AI 相关接口（问答 / Key 配置 / 说明书知识库）、RAG 检索链路（库表结构见 06） | 后端 |
| `06_数据库设计.md` | 全部数据库设计：实体关系、PostgreSQL 关系表建表 DDL、pgvector 说明书向量表 | 后端 |
| `CHANGELOG.md` | 版本变更记录：改了什么、影响哪个代码仓库 | 全员 |

**快速上手**：小程序与后端同学先看 01 + 02；建库 / 建模看 06；嵌入式同学看 01 + 03；做监护端实时推送看 04；做 AI 问答看 05。**接手新版本先看 `CHANGELOG.md`**。

---

## 文档如何同步到代码仓库（git subtree）

本仓库以 **git subtree** 单向同步到各代码仓库（Java 后端 / uni-app 小程序 / Web 管理后台）。

### 为什么用 subtree

- clone 后文档**就在项目树里**，不需要额外初始化步骤，人与 AI 都能直接读到（submodule 忘了 init 就是空目录）；
- 用 `--squash` 拉取时**不导入本仓库的历史**，每个代码仓库每次只多 1 个提交；
- 我们**不在代码仓库里改文档**（改文档走 Issue），所以 subtree 往回 push 麻烦这个缺点不适用。

### 首次接入（每个代码仓库做一次）

```bash
git remote add spec https://github.com/<you>/Interface-design-documentation.git
git subtree add --prefix=spec spec main --squash
```

### 更新（推荐交给 AI 助手执行）

```bash
git subtree pull --prefix=spec spec main --squash      # 拉最新
git subtree pull --prefix=spec spec v1.6 --squash      # 或按 tag 固定版本
git show --stat HEAD                                    # 看这次更新了哪些文件
```

### 约定

1. **`spec/` 目录只读**：不要在代码仓库里直接修改文档，否则下次 `subtree pull` 容易冲突；
2. **改文档走 GitHub Issues**：在本仓库提 Issue → 在本仓库修改并更新 `CHANGELOG.md` → 打版本 tag → 各代码仓库拉取；
3. **拉取时顺便读 CHANGELOG**：`CHANGELOG.md` 里写明"影响哪个代码仓库"，据此判断本地代码要不要跟着改；
4. 各代码仓库 README 中建议加一行说明：`spec/ 由文档仓库单向同步，请勿在此修改，问题请提到 <docs-repo>/issues`。
