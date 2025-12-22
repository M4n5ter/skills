# 配置与作用域

## 配置层级与覆盖顺序
从低到高依次覆盖：
1. 内置默认配置
2. 用户配置（User config）
3. 仓库配置（Repo config）
4. 工作区配置（Workspace config）
5. 命令行 `--config` 与 `--config-file`

## 用户配置位置
- 可选文件：`$HOME/.jjconfig.toml`
- 平台配置目录：`<platform>/jj/config.toml`
  - Linux：`$XDG_CONFIG_HOME` 或 `$HOME/.config`
  - macOS：`$HOME/Library/Application Support`
  - Windows：`%APPDATA%`
- 目录配置：`<platform>/jj/conf.d/*.toml`（按字母序合并）
- 可用命令查看实际路径：
  ```bash
  jj config path --user
  ```

## 仓库与工作区配置
- 仓库级：`.jj/repo/config.toml`
- 工作区级：`.jj/workspace-config.toml`

## 常用配置操作
- 编辑配置：
  ```bash
  jj config edit --user
  jj config edit --repo
  jj config edit --workspace
  ```
- 使用指定文件作为“基础配置”：
  ```bash
  JJ_CONFIG=/path/to/base-config.toml jj status
  ```
- 追加临时配置项：
  ```bash
  jj --config user.name="Jane Doe" --config user.email=jane@example.com status
  ```
- 追加配置文件：
  ```bash
  jj --config-file /path/to/extra-config.toml status
  ```

## 条件配置
- `--when` 允许在命令行指定条件配置（例如仅在某仓库生效）。
- `[[--scope]]` 与 `--when` 类似，可在配置文件中声明条件作用域。
  ```toml
  [[--scope]]
  --when.repositories = ["file:/path/to/repo1", "file:/path/to/repo2"]

  [ui]
  pager = "less -FRX"
  ```
