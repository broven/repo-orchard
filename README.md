# repo-orchard 🌳

**一个「壳子仓库」模板:把一组相关 git 仓库编排在一起,用 git worktree 并行开工——主要给 Code Agent 使用。**

## 这是什么(人读这一段就够)

跨多个仓库、又有多条并行任务时,单仓库的 `git worktree` 不够用。`repo-orchard` 提供一个「壳子」:
它登记一组相关仓库,每条任务开一个壳子 worktree,在里面把需要的几个仓库各拉一个 worktree 进来一起改,
改完各自开 PR,整组收工。适合「一条任务丢给一个 Code Agent」的并行开发。

---

> **以下面向 Code Agent。** 完整、权威的操作说明以 **[AGENTS.md](./AGENTS.md) 为唯一真相**;
> `CLAUDE.md` 仅转引它。本 README 补充「运行模型」与「编排工作流」两层背景,细节仍以 AGENTS.md 为准。

## 运行模型

- **壳子主 checkout** 只放编排工具(`wt` / `repos.toml`),**不在里面写业务代码**。
- **每条任务 = 壳子的一个 worktree**;worktree 目录名 = 工作区名。
- `./wt add <仓库>` 把选中仓库的 worktree 建到 `./repos/<名>/`,**分支统一用工作区名**,
  base 取该仓库 `origin/main`(拉取前先 `git fetch`,保证基于最新远端主分支)。
- `./repos/` 是运行时产物,已 gitignore。

```
repo-orchard/                    # 壳子主 checkout（工具之家）
repo-orchard-feature-login/      # ← 任务 A 的 worktree（分支 feature-login）
└── repos/{frontend,backend}/    #   两个子仓库各自的 worktree，都在 feature-login 分支
repo-orchard-hotfix-pay/         # ← 任务 B，与 A 完全隔离
└── repos/backend/
```

## 编排工作流(外层:怎么把任务分给 agent)

1. **给壳子开多个 worktree**,每个 = 一条并行任务,各起一个 Code Agent:
   ```bash
   git worktree add ../repo-orchard-<任务名> -b <任务名>
   ```
2. **告诉该 agent 这次要做什么** → agent 进入自己的 worktree,`./wt list` 读注册表,
   **自行判断需要哪些仓库** → `./wt add <仓库...>`。
3. agent 在 `./repos/*` 里**跨仓库改代码** → 完成后 `./wt pr` 给每个仓库开 PR。
4. **收工**:`./wt cleanup` 干净拆除子仓库 worktree → 再删掉这个壳子 worktree,整组结束。

多个壳子 worktree 相互隔离 → 天然支持多条任务、多个 agent 并行,互不踩踏。

## agent 在 worktree 内怎么操作

**一切以 [AGENTS.md](./AGENTS.md) 为准。** `wt` 命令速览:

| 命令 | 作用 |
|---|---|
| `./wt list` | 列出注册表所有仓库 + 介绍(据此选仓库) |
| `./wt add <名...>` | 建选中仓库的 worktree 到 `./repos/`,分支=工作区名 |
| `./wt status` | 已加入仓库 + 各自 git 状态 |
| `./wt pr [gh参数]` | 各仓库 `push` + `gh pr create --fill` |
| `./wt cleanup [--force]` | 拆除本工作区所有子仓库 worktree(有未提交改动会拒删) |
| `./wt prune` | 兜底:清所有仓库的悬空 worktree 记录 |

依赖:`bash` + `git`(`./wt pr` 需 `gh`)。

## 开一个新壳子

```bash
gh repo create my-xxx-orchard --template broven/repo-orchard --private --clone
cd my-xxx-orchard
# 编辑 repos.toml 填这组相关仓库（path/base/desc），即可按上面的流程开工
```
或在 GitHub 页面点 **Use this template**。

## 约定

- **`AGENTS.md` 唯一真相**;`CLAUDE.md` 只转引。
- **`repos.toml` 的 `base` 写 `origin/xxx`**(远程跟踪引用),才能保证从最新远端主分支拉。
- **`./repos/` 已 gitignore**;删工作区前先 `./wt cleanup`(忘了用 `./wt prune` 补救)。
