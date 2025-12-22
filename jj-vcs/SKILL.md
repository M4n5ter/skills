---
name: jj-vcs
description: 面向 Jujutsu(jj) 版本控制的使用、工作流、revset/fileset 语法、Git 互操作与配置排错指导。用于解答 jj 命令与概念差异、迁移 Git 流程到 jj、处理冲突/回滚、配置与远程书签相关问题。
---

# Jujutsu (jj) 使用

## 概览
用于在命令行和知识层面准确指导 Jujutsu(jj) 的日常操作、历史改写与 Git 互操作；必要时联动 `jj help` 与官方文档核验细节。

## 工作流决策树
1. 确认上下文
   - 获取版本与仓库类型：`jj --version`，判断纯 jj 仓库还是 Git backend/colocated。
   - 若执行过 `git clean -xdf`，`.jj/` 会被清理，需要重新 `jj git init`。
   - 明确目标：查看、创建/编辑、重写历史、冲突处理、Git 互操作、配置/排错。
2. 选择任务路径
   - 基础操作与改写：读取 `references/core-workflows.md`
   - 修订选择（revset）：读取 `references/revsets-filesets.md` 的 revset 章节
   - 文件集合（fileset）：读取 `references/revsets-filesets.md` 的 fileset 章节
   - 配置与作用域：读取 `references/config.md`
   - 远端与书签：明确 `bookmark` 与 `remote_bookmark` 的追踪关系
3. 输出时保持安全性
   - 明确可回滚路径：`jj op log`、`jj undo`、`jj op restore`
   - Git 混用场景说明 colocation 行为与 `jj git import/export` 机制
   - 远端历史改写前先创建备份书签

## 常见入口（只做索引）
```bash
jj st
jj log -r ::
jj diff --git
jj describe
jj new -m "message"
jj edit @
jj squash
jj rebase -s <revset> -d <revset>
jj split
jj bookmark list
jj bookmark create <name> -r <revset>
jj bookmark set <name> -r <revset> --allow-backwards
jj bookmark track <name>@<remote>
jj git clone <url>
jj git init
jj git fetch --remote <remote>
jj git push --remote <remote> --bookmark <name>
jj git remote add <name> <url>
jj git remote list
jj file untrack <paths...>
jj restore --from <revset>
jj op log
```

## Git 远端与书签操作
### 初始化与远端配置
```bash
jj git init
jj git remote add origin git@github.com:org/repo.git
jj bookmark track main@origin
```
```bash
jj config set --repo git.fetch origin
jj config set --repo git.push origin
```

### 提交并推送（最常用流程）
```bash
jj status
jj commit -m "message"
jj bookmark set main -r @-
jj git push --remote origin --bookmark main
```
- `jj commit` 会把工作区内容写入当前变更并创建一个新的空工作变更，所以需要用 `@-` 指向刚提交的那个变更。
- 若使用其他分支名，替换 `main` 即可。

### 同步与推送
```bash
jj git fetch --remote origin
jj git push --remote origin --bookmark main
```
- 如果提示 “Refusing to create new remote bookmark”，先执行：
```bash
jj bookmark track main@origin
```

### 覆盖远端历史（高风险）
```bash
jj bookmark set main -r <revset> --allow-backwards
jj git push --remote origin --bookmark main
```
- 建议先备份远端 `main`：
```bash
jj bookmark create backup-main -r main@origin
jj git push --remote origin --bookmark backup-main
```

### 从远端恢复工作区
```bash
jj restore --from main@origin
```

## 典型问题与排错
- 远端书签冲突：先 `jj bookmark track main@origin`，再根据冲突提示移动本地书签。
- 超大文件无法快照：优先加入 `.gitignore`，或临时提高阈值：
  - `jj config set --repo snapshot.max-new-file-size <bytes>`
- 嵌套仓库：在子目录执行 `git init`/`jj git init` 会产生嵌套仓库，需要删除子目录 `.git`。

## 交付原则
- 用最少命令达成目标；给出 1–2 条备选命令即可。
- 对不确定细节，优先引导 `jj help <cmd>` 或官方文档对应章节。
- 输出包含 revset/fileset 时，提醒用户使用引号避免 shell 解析问题。
