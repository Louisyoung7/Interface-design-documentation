# 家用智能药品箱 · 接口设计文档集（`medbox-spec`）

> 范围：Java 后端（Spring Boot）+ uni-app 小程序　|　协议：HTTP REST、MQTT（EMQX）、WebSocket　|　版本 V1.7
> 通信方式：当前全部为局域网（内网）通信，无公网域名。
> 变更记录见 `CHANGELOG.md`；同步到代码仓库的方式见文末《文档如何同步到代码仓库》。
>
> **本仓库是 `medbox-spec`**，通过 `git subtree` 与 `medbox-server`（后端）、`medbox-miniapp`（小程序）、`medbox-admin`（Web 后台）双向同步，在代码仓库中位于 `spec/` 目录。

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
| `07_特性分支清单_后端.md` | **Java 后端的特性分支待办清单**：每个待办 = 一个 `feat/xxx` 分支，前 8 项为地基，其余为可分配任务 | 后端 |
| `08_特性分支清单_小程序.md` | **uni-app 小程序的特性分支待办清单**：同样每个待办 = 一个分支，前 5 项为地基 | 小程序 |
| `CHANGELOG.md` | 版本变更记录：改了什么、影响哪个代码仓库 | 全员 |
| `AGENTS.md` | **给 AI 助手的工作约定**：特性分支流程、开工 pull / 完工 push 文档、远程更新 rebase、AI 的提醒义务 | AI 助手 / 全员 |

**快速上手**：小程序与后端同学先看 01 + 02；建库 / 建模看 06；嵌入式同学看 01 + 03；做监护端实时推送看 04；做 AI 问答看 05。**接手新版本先看 `CHANGELOG.md`**；**开始开发前先看 07 / 08 认领分支**。

---

## 分支与任务约定

- 一个功能 = 一个特性分支 = 07 / 08 清单里的一个 `- [ ]` 待办；分支名用清单中给出的 `feat/xxx`；
- 清单中 🧱 为**地基**（须先合并，后续任务都依赖它），🔧 为**可分配任务**；
- **修 Bug、重构项目结构不计入清单**，随所在模块一起提交；
- 完成后把待办勾成 `- [x]`，PR 描述写明"对应 07/08 清单第 N 项"；
- **AI 助手的完整工作流（含提醒义务）见 `AGENTS.md`**，尤其：开工前 `git subtree pull` 拉文档、完工时更新文档并 `git subtree push` 推回、远程更新时回主分支 fetch 再 rebase。

## 文档如何同步到代码仓库（git subtree）

本仓库以 **git subtree** 双向同步到各代码仓库（Java 后端 / uni-app 小程序 / Web 管理后台）。

### 为什么用 subtree

- clone 后文档**就在项目树里**，不需要额外初始化步骤，人与 AI 都能直接读到（submodule 忘了 init 就是空目录）；
- 用 `--squash` 拉取时**不导入本仓库的历史**，每个代码仓库每次只多 1 个提交；
- 支持 `git subtree push` **把改动推回本仓库**，文档与代码可以在同一个工作流里同步，不需要跳转 Issue。

### 首次接入（每个代码仓库做一次）

```bash
git remote add spec-repo https://github.com/<you>/medbox-spec.git
# 或 SSH：git remote add spec-repo git@github.com:<you>/medbox-spec.git
git subtree add --prefix=spec spec-repo main --squash
```

> 命名说明：**`spec/` 是文档目录，`spec-repo` 是远端仓库别名**，两者不要混淆。

### pull：开发前拉最新文档

```bash
git subtree pull --prefix=spec spec-repo main --squash  # 拉最新
git subtree pull --prefix=spec spec-repo v1.7 --squash  # 或按 tag 固定版本
git show --stat HEAD                                    # 看这次更新了哪些文件
```

### push：完成特性分支后把文档改动推回

```bash
# 1) 先拉最新文档（务必最先做，避免改在旧版本上、也避免与他人更新冲突）
git subtree pull --prefix=spec spec-repo main --squash

# 2) 在代码仓库里改 spec/ 下的文档（勾选 07/08 待办、补 CHANGELOG、修正接口/字段）
git add spec/ && git commit -m "docs(spec): <说明>"

# 3) 推回（推荐推到分支，由人在文档仓库合并）
git subtree push --prefix=spec spec-repo docs/<特性分支名>
# 小团队也可直接：git subtree push --prefix=spec spec-repo main
```

### 约定

1. **`spec/` 可以改，但要走 push 回传**：在代码仓库里改完文档后必须 `git subtree push` 推回本仓库，**不要只留在代码仓库里**，否则文档与代码会分叉；
2. **pull / push 都交给 AI 助手执行**，并在每次开工与收尾时由 AI 主动提醒（详见 `AGENTS.md`）；
3. **每次开工前先 pull**，完成后先更新文档再 push，避免文档落后于代码；
4. **拉取时顺便读 `CHANGELOG.md`**：其中写明"影响哪个代码仓库"，据此判断本地代码要不要跟着改；
5. 各代码仓库 README 中建议加一行说明：`spec/ 为接口文档（git subtree 同步），改动请通过 git subtree push 推回文档仓库`。

### AGENTS.md

本仓库根目录的 **`AGENTS.md`** 规定了给 AI 助手的完整工作流（特性分支、开工 pull、完工 push、远程更新时 rebase、以及 AI 的提醒义务）。各代码仓库请把它复制到**仓库根目录**一份，AI 工具会自动加载；内容与本仓库保持一致。
