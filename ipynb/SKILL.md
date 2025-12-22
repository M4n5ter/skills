---
name: ipynb-notebooks
description: 面向 .ipynb Notebook（Jupyter / JupyterLab / Google Colab / VS Code）的创建、审阅、重构与展示。涵盖工程化目录结构、token 高效处理、演示/分享模式、以及 uv/venv 可复现工作流。
---

## IPYNB Notebook（.ipynb）

### 概览

这个 skill 用于指导你以“工程化”的方式操作 `.ipynb` 文件与 notebook 项目（不限定 Jupyter，也适用于 Google Colab / VS Code Notebook 等环境）：

- **清晰的文件结构**：notebook 作为界面，逻辑沉到可复用的 `scripts/` 与 `lib/`
- **Token 高效工作流**：在 AI 读写 notebook 时尽量只读结构/代码，不读大输出
- **可展示模式**：用于 demo、团队共享、文档化的结构与输出规范
- **可复现环境**：优先使用 `uv`，或退回到 `venv`，确保可重复运行

### 适用场景

在以下场景使用本 skill：

- 新建 notebook 项目或单个 notebook
- 审阅 / 编辑已有 `.ipynb`（尤其是大文件、输出很多、diff 难读的情况）
- 整理 notebook 项目结构，把“可复用逻辑”从 notebook 抽到模块/脚本
- 为演示、分享、归档做“可跑通、可复现、可导出”的整理
- 改善 notebook 的长期可维护性与版本控制体验

### 核心原则

**Notebook 是界面（interface），不是库（library）。**

notebook 适合交互探索与叙事展示；可复用、可测试、可自动化的逻辑应放在：

- `scripts/`：可直接运行的脚本（不依赖 notebook UI）
- `lib/`：可复用模块（被 notebook 与脚本共同 import）

这样做带来的收益：

- 多 notebook 复用同一套逻辑
- 无需跑 notebook 就能测试关键逻辑
- 更容易在 CI/CD 中自动化执行（如导出、定时跑数）
- diff 更干净、版本控制更友好

### 快速上手

#### 新建一个 notebook 项目（推荐 uv）

1. **初始化项目（uv）**

   ```bash
   # Create project directory
   mkdir notebook-project && cd notebook-project

   # Initialize uv project
   uv init
   
   # Add dependencies (pick what you need)
   uv add jupyterlab pandas plotly
   ```

2. **建立目录结构**

   ```bash
   mkdir -p scripts lib data/{raw,processed} reports docs .archive
   touch data/.gitkeep data/raw/.gitkeep data/processed/.gitkeep reports/.gitkeep
   ```

3. **准备 `.gitignore`（示例）**

   ```gitignore
   # Virtual environments
   .venv/
   
   # Data and outputs (keep .gitkeep)
   data/**
   !data/**/
   !data/**/.gitkeep
   reports/**
   !reports/**/
   !reports/**/.gitkeep
   
   # Jupyter
   .ipynb_checkpoints/
   
   # Python
   __pycache__/
   *.pyc
   
   # Environment
   .env
   ```

4. **启动 notebook 环境**

   ```bash
   uv run jupyter lab
   ```

5. **需要更详细的模式时再加载引用文档：**

   - `references/file-structure.md`：目录结构与项目组织
   - `references/presentation-patterns.md`：演示/分享结构与输出规范
   - `references/token-efficiency.md`：AI 读写 notebook 的 token 高效策略

#### 审阅 / 对比一个已有 notebook（尽量只看结构与代码）

**推荐工作流：**

1. **先看结构，不读输出**

   ```bash
   # Cell types and counts
   jq '.cells | group_by(.cell_type) | map({type: .[0].cell_type, count: length})' notebook.ipynb
   
   # Code cells with outputs
   jq '[.cells[] | select(.cell_type == "code") | select(.outputs | length > 0)] | length' notebook.ipynb
   ```

2. **只对比代码 cell**

   ```bash
   # Extract code sources to compare
   jq '.cells[] | select(.cell_type == "code") | .source' notebook1.ipynb > /tmp/code1.json
   jq '.cells[] | select(.cell_type == "code") | .source' notebook2.ipynb > /tmp/code2.json
   diff /tmp/code1.json /tmp/code2.json
   ```

3. **确有必要再读取 notebook 正文**

   - 先明确要读哪一段、哪类 cell，再读
   - 大 notebook 优先按 cell 范围/主题分段读取
   - 细节见 `references/token-efficiency.md`

#### 整理一个 notebook 项目（抽逻辑、控输出、让它可复现）

目录组织建议见 `references/file-structure.md`。这里给一个可执行的最小迁移步骤：

1. 盘点根目录文件数量：`ls -1 | wc -l`
2. 移动脚本到 `scripts/`，文档到 `docs/`，旧 notebook 到 `.archive/`
3. 更新 notebook 中的 import：`from lib import module_name`
4. 验证仍可正常运行

### 可复现环境（uv / venv）

#### 为什么优先 uv？

uv 适合做以下事情：

- 快速、可复现的依赖管理
- 在项目依赖环境中运行工具（如 `jupyter`, `nbconvert`）
- 不污染全局 Python
- 跨平台一致性更好

#### 常用命令模式

**添加依赖：**

```bash
uv add plotly pandas duckdb
```

**安装工具（可选）：**

```bash
uv tool install jupyterlab
```

**在项目环境中运行：**

```bash
uv run jupyter lab
```

**单文件脚本声明依赖（用于 `uv run`）：**

```python
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "pandas",
#     "plotly",
# ]
# ///

import pandas as pd
import plotly.express as px

# Script code here
```

运行：`uv run script.py`

如果你不能使用 uv，也可以用 `python -m venv .venv` + `pip`，但要确保能一键复现（建议 `requirements.txt` 或 `pyproject.toml` + lockfile）。

### Token 高效工作流（面向 AI 与版本控制）

当通过 AI 助手读写 `.ipynb` 时：

#### 默认策略：提交前清理输出

**推荐 pre-commit：**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/kynan/nbstripout
    rev: 0.6.1
    hooks:
      - id: nbstripout
```

**确需保留输出时（不推荐）：**

```bash
SKIP=nbstripout git commit -m "Add notebook with visualization outputs"
```

更常见的做法是：输出落盘到 `reports/`，notebook 保持“可重新运行即可复现输出”（见 `references/token-efficiency.md`）。

#### 读之前先查询（结构优先）

**先看结构：**

```bash
jq '.cells | group_by(.cell_type) | map({type: .[0].cell_type, count: length})' notebook.ipynb
```

**只看代码：**

```bash
jq '.cells[] | select(.cell_type == "code") | .source' notebook.ipynb
```

#### 输出要“可控、可复现”

**倾向输出摘要，不要直接 dump 大对象：**

```python
print(f"[OK] Loaded {len(df_alarms):,} rows")
print(f"Columns: {', '.join(df_alarms.columns)}")
print(f"Date range: {df_alarms['timestamp'].min()} to {df_alarms['timestamp'].max()}")
```

**大输出落盘到文件：**

```python
fig.write_html(report_dir / "visualization.html")
print(f"[OK] Saved visualization to {report_dir}/visualization.html")
```

完整策略见 `references/token-efficiency.md`。

### 演示 / 分享模式

#### 推荐的 notebook 结构

1. **标题与概览** - 背景与目标
2. **准备** - 导入与配置
3. **数据加载** - 带反馈与错误处理
4. **摘要** - 高层统计
5. **可视化** - 带解释与使用提示
6. **结论** - 关键发现

#### 更“专业”的输出习惯

**统一状态输出：**

```python
print("[OK] Success")
print("[WARN] Warning")
print("[ERR] Error")
print("[INFO] Note")
```

**数字格式化：**

```python
print(f"Total: {count:,}")  # 2,055 instead of 2055
```

**按日期落盘到 reports：**

```python
from datetime import datetime

today = datetime.now().strftime('%Y-%m-%d')
report_dir = Path("reports") / today
report_dir.mkdir(parents=True, exist_ok=True)

fig.write_html(report_dir / "chart.html")

latest = Path("reports/latest")
if latest.exists():
    latest.unlink()
latest.symlink_to(today, target_is_directory=True)
```

完整模式与模板见 `references/presentation-patterns.md`。

### 资源索引

#### references/file-structure.md

包含：

- 推荐目录结构
- 文件组织规则与命名约定
- Git 友好（ignore、diff、清理输出）
- 现有项目迁移步骤
- 示例结构

**适合在：** 新建项目、重构目录、统一约定时加载。

#### references/token-efficiency.md

包含：

- 输出清理与版本控制策略
- 不读输出的结构化查询方法
- 大 notebook 的分段读取与 diff 思路
- 常用 `jq` / CLI 模式
- cell 输出管理

**适合在：** 需要省 token、要审阅大 notebook、要做自动化处理时加载。

#### references/presentation-patterns.md

包含：

- 演示型 notebook 的结构模板
- 可读性与叙事节奏
- 交互元素与可导出策略
- 错误处理与可复现检查点
- Markdown / Code cell 分工
- 导出 HTML/PDF 的注意事项

**适合在：** 做 demo、团队分享、发布文档前加载。

### 最佳实践速记

1. **结构**：notebook 作为界面，逻辑下沉到 `scripts/` / `lib/`
2. **依赖**：优先 uv，保证一键复现
3. **版本控制**：默认清理输出（pre-commit/nbstripout/nbconvert）
4. **省 token**：先查询结构再阅读；大输出落盘
5. **展示**：叙事清晰、输出克制、错误处理明确
6. **可复现**：确保 “Restart & Run All” 能跑通
7. **数据流**：raw → processed → reports
8. **Git 友好**：忽略数据与产物，保留目录骨架（`.gitkeep`）

### 示例流程

```bash
# 1. Create project
mkdir my-analysis && cd my-analysis
uv init
uv add jupyterlab pandas plotly

# 2. Set up structure
mkdir -p scripts lib data/{raw,processed} reports
touch data/.gitkeep data/raw/.gitkeep data/processed/.gitkeep reports/.gitkeep

# 3. Create notebook
uv run jupyter lab

# 4. As you work:
# - Keep logic in lib/ and scripts/
# - Save outputs to reports/ with dates
# - Keep outputs minimal
# - Strip outputs before committing

# 5. Before presenting:
# - Run "Restart & Run All" to test
# - Add context and documentation
# - Consider exporting to HTML
jupyter nbconvert --to html --execute notebook.ipynb
```

### 速查表

**目录组织：**

- Notebook：项目根目录（或按规模拆到 `notebooks/`）
- 脚本：`scripts/`
- 模块：`lib/`
- 数据：`data/raw/`, `data/processed/`
- 报告：`reports/YYYY-MM-DD/`
- 归档：`.archive/`

**uv 常用命令：**

- `uv init`：初始化项目
- `uv add <package>`：添加依赖
- `uv run <command>`：在项目环境中运行命令
- `uvx <tool>`：运行临时工具（不写入项目依赖）

**省 token：**

- 清理输出：pre-commit hook，或 `jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace notebook.ipynb`
- 查询结构：`jq '.cells | group_by(.cell_type)'`
- 对比代码：`jq '.cells[] | select(.cell_type == "code") | .source'`

**展示：**

- 数字格式化：`{count:,}`
- 按日期落盘：`reports/YYYY-MM-DD/`
- 执行验证：`jupyter nbconvert --execute`
