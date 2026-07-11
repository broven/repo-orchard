# repo-orchard 🌳

**一个「壳子仓库」模板：把几个相关的 git 仓库编排在一起，用 git worktree 的方式并行开工。**

单个仓库的 `git worktree` 只能隔离一个仓库。但很多任务是**跨多个仓库**的（前端 + 后端 + 共享库…），
而且你常常有**多条并行任务**同时进行。`repo-orchard` 就是为这个场景做的壳子：

- 一个壳子仓库登记「一组相关仓库」（`repos.toml`）；
- 每开一条任务，就给壳子开一个 **worktree**，在里面 `./wt add` 把需要的仓库各拉一个 worktree 进来；
- 于是「一个工作区目录」里同时装着这次任务要动的几个仓库，改完各自开 PR，收工整组拆掉。

---

## ⭐ 关键：这个壳子是「用 worktree 的方式」使用的

**不要直接在壳子仓库的主 checkout 里干活。** 主 checkout 只用来放 `wt`、`repos.toml` 这套工具本身。

真正干活的姿势是——**每条任务开一个壳子的 worktree**：

```
repo-orchard/                    # 壳子主 checkout（工具之家，别在这写业务代码）
├── wt / repos.toml / AGENTS.md  # 编排工具与注册表

repo-orchard-feature-login/      # ← 任务 A 的 worktree（分支 feature-login）
└── repos/
    ├── frontend/                #   frontend 仓库的 worktree（分支 feature-login）
    └── backend/                 #   backend  仓库的 worktree（分支 feature-login）

repo-orchard-hotfix-payments/    # ← 任务 B 的 worktree（分支 hotfix-payments），与 A 完全隔离
└── repos/
    └── backend/
```

- **工作区名 = 壳子 worktree 的目录名**，也就是各子仓库新建分支的名字。
- 每个子仓库的 worktree 从它自己的 `origin/main`（拉取前会 `git fetch`）新建同名分支，保证基于最新主分支。
- 多个壳子 worktree 之间互不干扰 → 天然支持多条并行任务，很适合一条任务丢给一个 Code Agent。

> 搭配 [Orca](https://onorca.dev)、`git worktree`、或任何 worktree 管理器都行——壳子本身不绑定任何工具。

---

## 🚀 快速开始

1. **用本模板建一个你自己的壳子**（GitHub 上点 **Use this template**，或用 CLI）：
   ```bash
   gh repo create my-project-orchard --template broven/repo-orchard --private --clone
   cd my-project-orchard
   ```

2. **登记你的相关仓库**：编辑 `repos.toml`，把示例换成你本机的仓库路径与介绍：
   ```toml
   [frontend]
   path = "/Users/you/code/frontend"
   base = "origin/main"
   desc = "Web 前端（React），主域 app.example.com"

   [backend]
   path = "/Users/you/code/backend"
   base = "origin/main"
   desc = "API 后端（Go），网关 + 鉴权"
   ```

3. **开一条任务的工作区**（给壳子开个 worktree）：
   ```bash
   git worktree add ../my-project-orchard-feature-login -b feature-login
   cd ../my-project-orchard-feature-login
   ```

4. **拉需要的仓库、干活、开 PR、收工**：
   ```bash
   ./wt list                 # 看有哪些仓库
   ./wt add frontend backend # 各拉一个 worktree 到 ./repos/，分支=feature-login
   # ... 在 ./repos/frontend、./repos/backend 里改代码 ...
   ./wt pr                   # 各仓库 push 并开 PR
   ./wt cleanup              # 收工：干净拆除子仓库 worktree（删本工作区前必做）
   ```

---

## 🛠 `wt` 命令

| 命令 | 作用 |
|---|---|
| `./wt list` | 列出注册表里所有仓库 + 介绍 |
| `./wt add <名> [名...]` | 把选中仓库的 worktree 建到 `./repos/`，分支 = 工作区名（base 取 `origin/main`，建前先 fetch） |
| `./wt status` | 当前工作区已加入的仓库 + 各自 git 状态 |
| `./wt pr [gh参数]` | 对每个已加入仓库 `push` 并 `gh pr create --fill` |
| `./wt cleanup [--force]` | 拆除本工作区所有子仓库 worktree（有未提交改动会拒删，`--force` 强拆） |
| `./wt prune` | 兜底：扫描注册表所有仓库，清掉悬空 worktree 记录（忘了 cleanup 就直接删目录时用） |

依赖：`bash` + `git`（`./wt pr` 需要 `gh`）。零其它依赖。

---

## 📎 约定

- **`AGENTS.md` 是唯一真相**：Code Agent 的完整操作说明在那里；`CLAUDE.md` 只是转引它。
- **`repos.toml` 的 `base` 写 `origin/xxx`**（远程跟踪引用），才能保证从最新远端主分支拉。
- **`./repos/` 已 gitignore**：子仓库 worktree 是运行时产物，不进壳子仓库的版本库。
- **删工作区前先 `./wt cleanup`**：否则源仓库会残留悬空 worktree 记录（可用 `./wt prune` 补救）。
