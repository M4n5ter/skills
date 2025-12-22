# 核心概念与工作流

## 1. 核心概念
- 以“变更（change）”为中心：每个变更有 change id，提交哈希会随着改写而变化。
- 工作副本对应一个提交：`@` 表示当前工作副本提交，`@-` 表示其父提交。
- Git backend 场景下，Git 的暂存区会被忽略，查看差异以 `jj diff` 为准。

## 2. 初始化与克隆
- 克隆 Git 仓库：
  ```bash
  jj git clone <url> [<dest>] [--remote <remote-name>]
  ```
- 新建 Git backend 仓库（默认 colocated）：
  ```bash
  jj git init <name>
  ```
- 基于现有 Git 仓库初始化：
  ```bash
  jj git init --git-repo <path> <name>
  ```
- 禁用 colocated：
  ```bash
  jj git init --no-colocate <name>
  ```
- Colocation 管理：
  ```bash
  jj git colocation status
  jj git colocation enable
  jj git colocation disable
  ```

## 3. 查看与描述
- 查看状态与历史：
  ```bash
  jj st
  jj log -r ::
  ```
- 查看差异：
  ```bash
  jj diff
  jj diff --git
  ```
- 设置或编辑变更描述：
  ```bash
  jj describe
  ```

## 4. 创建/编辑变更
- 结束当前变更并开始新变更：
  ```bash
  jj new
  jj new -m "message"
  ```
- 切换到某个变更继续编辑：
  ```bash
  jj edit <revset>
  ```
- 将工作副本内容并入父变更（类似 amend）：
  ```bash
  jj squash
  ```
- 取消跟踪文件（配合忽略规则）：
  ```bash
  jj file untrack <path>
  ```
- 查看变更演化历史：
  ```bash
  jj evolog
  ```

## 5. 重写与移动内容
- 变基（包含后代变更）：
  ```bash
  jj rebase -s <source> -o <destination>
  ```
- 拆分变更（fileset 决定拆分内容）：
  ```bash
  jj split
  ```

## 6. 冲突处理
- 冲突是一等公民：冲突可以存在于提交中，不会阻塞后续操作。
- 常见流程（从冲突提交中分支修复）：
  ```bash
  jj new <conflicted>
  jj resolve
  jj diff
  jj squash
  ```
- 冲突标记样式由 `ui.conflict-marker-style` 控制（`snapshot` 或 `git`）。
- 冲突在 `jj show`、`jj diff` 输出中会被具体化展示。

## 7. 书签与远程
- 书签类似分支，但没有“当前书签”的概念：
  ```bash
  jj bookmark list
  jj bookmark <subcommand>
  ```
- 在书签上继续工作：
  ```bash
  jj new <bookmark>
  ```
- 推送书签到 Git：
  ```bash
  jj git push --bookmark <name>
  ```
- 远程书签与跟踪：
  ```bash
  jj bookmark track <name@remote>
  jj bookmark untrack <name@remote>
  jj bookmark list --tracked
  ```
- 自动跟踪配置：
  ```toml
  [remotes.origin]
  auto-track-bookmarks = true
  ```

## 8. 操作日志与回滚
- 操作日志与撤销：
  ```bash
  jj op log
  jj undo
  jj op revert <op>
  jj op restore <op>
  ```
- 操作选择与回放：
  ```bash
  jj log --at-op <op>
  ```
- 操作表达式要点：
  - `@` 表示当前操作，`x-` 表示父操作，`x+` 表示子操作。
  - 可用 `--at-op` / `--at-operation` 在指定操作上执行命令。

## 9. Git 互操作要点
- colocated 模式下，`jj` 与 `git` 共享工作区，`jj` 会自动导入/导出变更。
- `git` 命令可能导致 HEAD 处于 detached 状态；必要时使用 `git switch` 重新附着。
- 非 colocated Git backend 时，使用：
  ```bash
  jj git import
  jj git export
  ```
