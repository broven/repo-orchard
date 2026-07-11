# repo-orchard 🌳

**English** | [中文](./README.zh-CN.md)

## What is this

A **shell-repo template** that groups **a set of related repositories** together and lets you work across them in parallel with git worktrees — built primarily for Code Agents.

When your work spans multiple repos and you have several tasks in flight, a single-repo `git worktree` isn't enough. With repo-orchard you open one "workspace" per task, and it automatically assembles the specific repos that task touches, so a Code Agent can edit across them and open a PR in each. **The end result: multiple cross-repo tasks running at once without stepping on each other — just hand one task to one agent.**

## Quick start (3 steps)

1. **Make the template your own** — either:
   - Click **Use this template** on GitHub to create a new repo; or
   - One command locally: `gh repo create my-xxx-orchard --template broven/repo-orchard --private --clone`

2. **Configure your repos** — edit `repos.toml` and add the **local path** and a **one-line description** for each repo you want to work on together (follow the examples in the file).

3. **Open your Code Agent** — start your Code Agent in a workspace and give it the task. The rest (pick repos, add worktrees, open PRs, clean up) it handles itself per `AGENTS.md`.

> The detailed workflow and commands live in [`AGENTS.md`](./AGENTS.md) — that's the single source of truth written for the Code Agent; humans don't need to read it.
