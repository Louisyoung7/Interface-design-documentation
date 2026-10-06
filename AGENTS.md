# AGENTS.md — 给 AI 助手的工作约定

> 适用范围：本项目三个代码仓库（**`medbox-server` 后端 / `medbox-miniapp` 小程序 / `medbox-admin` Web 后台**）以及文档仓库 **`medbox-spec`** 本身。
> 代码仓库中，接口文档位于 **`spec/`** 目录（由文档仓库通过 `git subtree` 同步）。
> **AI 助手在动手写代码之前，必须先读完本文件，再读 `spec/00_README_文档索引.md`。**
> 本文件在文档仓库根目录有一份；代码仓库建议把它复制到**仓库根目录**一份（AI 工具会自动加载），内容保持一致即可。
> **在文档仓库里工作时不需要 subtree**：直接改文件并提交即可；下面的 pull / push 流程只在**代码仓库**里执行。

---

## 0. 项目一句话

家用智能药品箱：设备（药箱主控 + 传感器 + 摄像头）经 **MQTT(EMQX)** 上报，Java 后端落库并判定，**WebSocket** 推给 uni-app 小程序，**HTTP REST** 供小程序读写，AI 问答走大模型（OCR 与 Embedding 本地化）。当前全部为**局域网**通信。

---

## 1. 铁律（AI 发现违反时，必须先提醒用户再继续）

1. **绝不在主分支开发**：一切改动都在 `feat/xxx` 特性分支上做。分支名取自 `spec/07` 或 `spec/08` 清单。
2. **每次开工前先拉最新文档**：`git subtree pull --prefix=spec spec-repo main --squash`，再对照文档实现。
3. **完成特性分支、合并前必须更新文档并推回文档仓库**：勾选 07/08 待办、补 `CHANGELOG.md`、修正与实现不一致的接口/字段。
4. **远程有更新时用 rebase，不要 merge**：切回主分支 fetch/pull，再回特性分支 `git rebase main`。
5. **修 Bug / 重构不单独开分支记账**，但仍要走特性分支，随所在模块一起提交。

---

## 2. 开始开发前（每次会话都要做）

```bash
# ① 确认自己在特性分支上（不是 main）
git branch --show-current

# ② 拉最新文档（在特性分支上执行即可）
git subtree pull --prefix=spec spec-repo main --squash

# ③ 看这次文档有没有变
git show --stat HEAD
```

然后读文档：

- 后端任务 → `spec/07_特性分支清单_后端.md` 找到自己那条待办，按其"参考"列读 `spec/02`（接口）、`spec/06`（表结构）、`spec/03`（MQTT）；
- 小程序任务 → `spec/08_特性分支清单_小程序.md`，读 `spec/02`、`spec/04`；
- 不确定口径 → 查 `spec/01`（时区、权限、过期、鉴权等全局口径）。

**提醒义务**：如果用户直接在主分支上让我改代码，或本次会话还没拉过文档，我必须先说一句：

> ⚠️ 当前在 `main` 分支 / 尚未同步最新文档，建议先 `git switch -c feat/xxx` 并 `git subtree pull --prefix=spec spec-repo main --squash`，需要我帮你做吗？

---

## 3. 开发中

- 严格按文档实现：**字段名、错误码、枚举值、Topic、Payload、时区口径**以文档为准；
- 发现文档有歧义或与实际不符：**不要擅自偏离文档**，把问题记下来，在第 4 步"更新文档"时一并修正并说明；
- 新增接口 / 字段 / 表 → 同步改 `spec/02` / `spec/06` / `spec/03`，不要只在代码里加；
- 不确定时先问用户，不要猜。

---

## 4. 完成一个特性分支时（合并前必做）

```bash
# ① 先拉最新文档（务必最先做，避免改在旧版本上、也避免与他人更新冲突）
git subtree pull --prefix=spec spec-repo main --squash

# ② 更新 spec/ 下的文档（勾选待办、补 CHANGELOG、修正接口/字段）
#    - 后端：spec/07 对应项改为 - [x]
#    - 小程序：spec/08 对应项改为 - [x]
#    - 在 spec/CHANGELOG.md 追加/补充本次改动与影响的代码仓库

# ③ 在代码仓库提交这些文档改动
git add spec/
git commit -m "docs(spec): 更新 <模块> 接口/字段，勾选 07/08 第 N 项"

# ④ 推回文档仓库（推荐推到分支，再由人在文档仓库合并）
git subtree push --prefix=spec spec-repo docs/<特性分支名>
#    小团队也可直接：git subtree push --prefix=spec spec-repo main
```

**提醒义务**：如果用户说"这个分支做完了 / 帮我合并"，而 `spec/` 还没有文档改动，我必须先提醒：

> ⚠️ 合并前请先更新 `spec/` 文档（勾选 07/08 待办 + 补 CHANGELOG）并 `git subtree push` 推回文档仓库，否则文档会落后于代码。需要我来做吗？

---

## 5. 远程仓库有更新时（同步主线）

```bash
git stash -u                 # 有未提交改动先收起来
git switch main
git pull                     # 或 git fetch && git merge --ff-only
git switch <特性分支>
git rebase main              # 用 rebase 保持线性历史
# 冲突：解决后 git add <file> && git rebase --continue
git push --force-with-lease  # 分支已推送过时用，不要用 --force
git stash pop                # 恢复未提交改动
```

**提醒义务**：用户说"拉取一下最新代码 / 远程更新了"时，提醒按上面流程走（**先回主分支 fetch，再回特性分支 rebase**），不要直接在特性分支上 `git pull origin main`（会产生合并提交）。

---

## 6. 常用命令速查

| 场景 | 命令 |
|------|------|
| 新建特性分支 | `git switch -c feat/xxx main` |
| 开发前同步文档 | `git subtree pull --prefix=spec spec-repo main --squash` |
| 看本次文档改了什么 | `git show --stat HEAD` |
| 文档改动推回 | `git subtree push --prefix=spec spec-repo docs/<分支名>` |
| 同步主线 | `git switch main && git pull && git switch - && git rebase main` |
| 首次接入文档（一次性） | `git remote add spec-repo <文档仓库URL>` + `git subtree add --prefix=spec spec-repo main --squash` |

> 命名说明：`spec/` 是**文档目录**，`spec-repo` 是**远端仓库别名**，两者不要混淆。

---

## 7. 自检清单（每次回答前过一遍）

- [ ] 我这次改的是**特性分支**，不是 `main`？
- [ ] 本次会话**拉过最新文档**了吗？
- [ ] 实现与 `spec/` 里的字段 / 错误码 / 枚举**一致**吗？
- [ ] 改了接口或表结构 → **`spec/02` / `spec/06` / `spec/03` 同步改了**吗？
- [ ] 功能已完成 → **勾选了 07/08 待办 + 补了 CHANGELOG** 吗？
- [ ] 用户要合并 → 提醒过"先更新并 push 文档"吗？
- [ ] 用户要拉远程更新 → 提醒过"回主分支 fetch + 特性分支 rebase"吗？
