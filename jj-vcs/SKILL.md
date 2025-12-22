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
   - 明确目标：查看、创建/编辑、重写历史、冲突处理、Git 互操作、配置/排错。
2. 选择任务路径
   - 基础操作与改写：读取 `references/core-workflows.md`
   - 修订选择（revset）：读取 `references/revsets-filesets.md` 的 revset 章节
   - 文件集合（fileset）：读取 `references/revsets-filesets.md` 的 fileset 章节
   - 配置与作用域：读取 `references/config.md`
3. 输出时保持安全性
   - 明确可回滚路径：`jj op log`、`jj undo`、`jj op restore`
   - Git 混用场景说明 colocation 行为与 `jj git import/export` 机制

## 常见入口（只做索引）
```bash
jj st
jj log -r ::
jj diff --git
jj describe
jj new -m "message"
jj edit @
jj squash
jj rebase -s <revset> -o <revset>
jj split
jj bookmark list
jj git clone <url>
jj op log
```

## 交付原则
- 用最少命令达成目标；给出 1–2 条备选命令即可。
- 对不确定细节，优先引导 `jj help <cmd>` 或官方文档对应章节。
- 输出包含 revset/fileset 时，提醒用户使用引号避免 shell 解析问题。
