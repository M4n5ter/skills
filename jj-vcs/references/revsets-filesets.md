# Revset 与 Fileset

## Revset（选择修订集）

### 基本定位
- revset 是一个函数式语言，用于选择一组修订。
- 多数命令接受 revset；有些命令需要单个修订，表达式需能解析为单一目标。
- 默认只可见“可见修订”，如需包含隐藏修订，使用 `all()`。

### 常用方式
```bash
jj log -r ::           # 所有可见修订
jj log -r 'all()'      # 包含隐藏修订
```

### 字符串字面量
- 字符串使用单引号或双引号。
- 支持反斜杠转义。
- 使用 `r'...'` 或 `r"..."` 表示 raw string。

### Symbols 与优先级
- `@`：当前工作副本提交。
- `<workspace>@`：指定工作区的工作副本提交。
- `<name>@<remote>`：远程书签。
- 直接使用 commit id 或 change id。
- 名称歧义时，使用 `commit_id()` 或 `change_id()` 显式指定。
- Symbols 优先级：tag > bookmark > git ref > commit id > change id。

### 操作符（优先级从高到低）
- 祖先/后代与范围：
  - `x-`, `x+`
  - `x::`, `x..`, `::x`, `..x`, `x::y`, `x..y`, `::`, `..`
- 一元与集合运算：
  - `~x`, `x & y`, `x ~ y`, `x | y`

## Fileset（选择文件集合）

### 基本定位
- fileset 描述一组文件，常用于 `jj split`、`jj describe --from` 等命令。
- shell 会先展开通配符，建议对 fileset 参数加引号。

### 字符串字面量
- 规则与 revset 相同：单双引号、反斜杠转义、raw string。

### Patterns
- `file:pattern`
- `glob:pattern`
- `prefix-glob:pattern`
- `root:path`
- `root-file:path`
- `root-glob:pattern`
- `root-prefix-glob:pattern`

### 操作符与函数
- 操作符：`x & y`, `x ~ y`, `x | y`, `~x`
- 函数：`all()`, `none()`

### 示例
```bash
jj split main -i
jj describe --reset-author --author "bar@example.com" --from "root:file2"
```
