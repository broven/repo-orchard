# repo-orchard 🌳

[English](./README.md) | **中文**

## 这是什么

把**一组相关的仓库**编排在一起、用 git worktree **并行开工**的壳子模板——主要给 Code Agent 使用。

跨多个仓库、又有多条并行任务时,单仓库的 `git worktree` 不够用。用 repo-orchard,你为每条任务开一个「工作区」,
里面自动装着这次要改的那几个仓库,让 Code Agent 跨仓库一起改、各自开 PR。**最终效果:多条跨仓库任务同时进行、互不干扰,一条任务丢给一个 agent 就行。**

## 使用概览(三步)

1. **把模板变成你自己的仓库** —— 二选一:
   - GitHub 上点 **Use this template** 新建一个仓库;或
   - 本地一条命令:`gh repo create my-xxx-orchard --template broven/repo-orchard --private --clone`

2. **配置你的仓库** —— 编辑 `repos.toml`,把这次要一起工作的几个仓库的**本地路径**和**一句话说明**填进去(照文件里的示例改)。

3. **打开 Code Agent** —— 在工作区里启动你的 Code Agent,把任务交给它。剩下的(选仓库、拉 worktree、开 PR、收工)它会照 `AGENTS.md` 自己完成。

> 更细的操作方式与命令都在 [`AGENTS.md`](./AGENTS.md) 里,那是给 Code Agent 读的唯一真相,人类不用关心。
