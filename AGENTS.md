# 多仓库工作区壳子（给 Code Agent 的说明）

这个仓库是一个**壳子 / 编排中枢**，本身几乎不装业务代码。
你现在所在的目录 = 一个**工作区**（一条并行任务）。你的活儿通常横跨多个 git 仓库，
本仓库提供一个 helper `./wt`，让你把需要的仓库各开一个 worktree 到本工作区下面来干活。

## 核心概念

- **工作区名** = 当前目录名（也就是本 worktree 的名字），例如 `feature-login`。
- `./wt add` 会把选中仓库的 worktree 建到 `./repos/<仓库名>/`，**分支统一用工作区名**。
- 所有仓库信息登记在 `repos.toml`（名字 / 路径 / base / 一句话介绍）。
- `./repos/` 是运行时生成、已 gitignore，不会污染本壳子仓库。

## 标准流程

1. **看有哪些仓库**：
   ```bash
   ./wt list
   ```
   根据每个仓库的介绍，判断本次任务需要哪几个。

2. **把需要的仓库加进来**（可一次多个）：
   ```bash
   ./wt add repo-a repo-b
   ```
   完成后 `./repos/repo-a`、`./repos/repo-b` 就是对应仓库、位于工作区同名分支上的 worktree。进入目标仓库后，先执行下一步，再开始工作。

3. **读取目标仓库说明（必须）**：

   `./wt add` 完成后，在读取业务代码、安装依赖、启动服务或修改文件之前，必须先检查并完整阅读每个目标仓库根目录中的：

   - `AGENTS.md`；
   - `ONBOARD.md`（若存在）；
   - `mise.toml` / `.mise.toml`（若存在，优先使用其中定义的任务）。

   随后继续查找目标子目录内更具体的 `AGENTS.md`。目录层级更深的说明优先适用于该目录。禁止在未完成上述检查时自行猜测安装、启动或测试命令。

4. **干完开 PR**（进各子仓库自己 push + 开 PR，不走 wt 封装）：
   ```bash
   cd repos/<仓库名>
   git push -u origin HEAD          # 分支名即工作区名
   gh pr create --fill              # base 默认取仓库默认分支；多个仓库就逐个来
   ```

5. **收工拆除**（把子仓库 worktree 干净移除，避免源仓库残留悬空 worktree）：
   ```bash
   ./wt cleanup
   ```
   若某仓库还有未提交改动会被拒绝删除并提示；确认无价值后可 `./wt cleanup --force`。
   > **回收外部资源**：`./wt cleanup` 在删每个 worktree **之前**，会自动在该 worktree
   > 目录内跑一次它声明的 `mise run teardown`（若定义了），回收 dev 期间拉起的
   > **per-worktree 外部资源**（典型如 docker 起的 DB/缓存容器和卷）。必须「删之前、在
   > worktree 里」跑，因为这类资源名按当前分支算（如 compose project `<repo>-<slug>`），
   > worktree 一删就对不上、只能人肉逐个清。teardown 是 best-effort：仓库没定义就跳过，
   > 失败只告警、不阻断拆除。**约定见下方「让仓库能被自动清理」。**
   > **重要**：删除本工作区（这个 worktree 目录）之前，**务必先 `./wt cleanup`**。
   > 直接删目录会把 `./repos/` 下的子 worktree 一起强删，源仓库会残留指向已消失路径的
   > 悬空记录。（用 Orca 等工具「删除 worktree」也会 rm -rf 目录，同样要先 cleanup；
   > 若把 archive/pre-remove 钩子接到 `./wt cleanup`，上面的 teardown 也一并自动生效。）

## 兜底：忘了 cleanup 怎么办

如果哪次直接删了工作区、忘了先 cleanup，源仓库会残留悬空 worktree 记录。到对应源仓库里跑一次 `git worktree prune` 清掉即可：
```bash
git -C /path/to/repo worktree prune -v   # 按 repos.toml 里各仓库 path 逐个清
```
（`wt` 不再封装此步——直接用 git 原生命令，少一层维护。）

## 让仓库能被自动清理（约定：`mise run teardown`）

如果某个子仓库在 **dev 时会拉起 per-worktree 的外部资源**——docker 起的 DB/缓存容器与
命名卷、临时云沙箱、后台常驻进程等——那它**应当**在自己的 mise 配置里定义一个名为
`teardown` 的任务，把这些资源**按当前 worktree/分支精确回收**。

- **命名固定**：任务名就叫 `teardown`（`./wt cleanup` 靠这个名字探测；只有仓库确实
  定义了才会调）。
- **要做到 per-worktree 精确回收**：teardown 里用 `git rev-parse --abbrev-ref HEAD`
  之类算出本 worktree 独有的资源名（如 `docker compose -p <repo>-<slug> ... down -v`），
  只清本 worktree 拉起的那套，不误伤别的并行 worktree。
- **幂等 + best-effort**：没起过资源也能安全空跑；失败不应炸（`./wt cleanup` 只告警不阻断）。
- **语义 = 彻底销毁本 worktree 的 dev 环境**：该删的容器/卷一并删（不是「留数据下次
  秒起」那种 stop）。

具备该任务后，`./wt cleanup`（以及任何接到它的 archive/pre-remove 钩子）拆 worktree 前
都会自动回收，无需再人肉记着清 docker。**没有这类外部资源的仓库（纯文档/纯库）不用定义。**

## 加一个新仓库到注册表

编辑 `repos.toml`，复制一段改三行：
```toml
[新名字]
path = "/abs/path/to/repo"
base = "origin/main"   # 可省略，自动探测
desc = "一句话介绍"
```

## 维护介绍：让下次更好找（重要）

`repos.toml` 里的 `desc` 是**预设**，不一定准。如果你在挑仓库时遇到下面任一情况，
**顺手回去改一下对应的 `desc`**，让同类需求下次一眼命中：

- 按 `desc` 选了仓库，结果发现选错了（内容跟介绍不符）；
- 用户的说法（功能名、域名、俗称…）在现有 `desc` 里找不到对应仓库；
- 费了很大劲、绕了很大弯（翻代码 / 试了好几个仓库）才定位到正确的仓库。

改的时候补上「让你多花力气的那个关键线索」——比如真实用途、关键域名、常用别名/俗称、
主要技术栈或关键子目录。**别过拟合**：只加能稳定复用的信息，不要把某次任务的临时细节堆进去。
**单条 `desc` 最多 200 字**，超了就提炼精简，不要越写越长。
