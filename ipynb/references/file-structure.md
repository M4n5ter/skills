# 目录结构与项目组织（.ipynb）

本文件聚焦于：让 `.ipynb` 在 Jupyter / JupyterLab / Google Colab / VS Code Notebook 等环境中都更容易复现、审阅、迁移与维护。

## 目标

- notebook 负责“叙事 + 交互”，业务逻辑沉到可复用的 `lib/` 与可运行的 `scripts/`
- 输出与产物默认落盘到 `reports/`，而不是塞进 cell output
- `data/` 与 `reports/` 默认不进 Git，但保留目录骨架（`.gitkeep`）
- 启动位置与路径策略清晰，避免“换个环境就找不到文件”

## 推荐目录结构

### 方案 A：小型项目（notebook 放根目录）

适合 notebook 数量不多（例如 1–5 个），且你希望打开仓库就能看到主入口。

```text
project/
  README.md
  pyproject.toml
  uv.lock
  analysis.ipynb
  lib/
    __init__.py
    io.py
    features.py
  scripts/
    run_pipeline.py
  data/
    .gitkeep
    raw/
      .gitkeep
    processed/
      .gitkeep
  reports/
    .gitkeep
  docs/
  .archive/
```

### 方案 B：中大型项目（notebooks/ 子目录）

适合 notebook 数量变多、主题变杂时，把 notebook 统一放到 `notebooks/`，并用子目录分层。

```text
project/
  README.md
  pyproject.toml
  uv.lock
  notebooks/
    00-overview.ipynb
    exploration/
      2025-12-17-eda.ipynb
    reports/
      2025-12-17-summary.ipynb
  lib/
    __init__.py
    io.py
    features.py
  scripts/
    run_pipeline.py
  data/
    .gitkeep
    raw/
      .gitkeep
    processed/
      .gitkeep
  reports/
    .gitkeep
  docs/
  .archive/
```

## 关键约定

### 1) Notebook 命名与分层

- 优先用“序号 + 主题”或“日期 + 主题”命名，避免 `Untitled42.ipynb`
- 明确区分：
  - `exploration/`：探索型（允许临时性代码，但最终要沉淀到 `lib/`）
  - `reports/`：结论型（叙事清晰、可复现、输出克制、可导出）
- 一旦 notebook 对外分享或作为“交付物”，默认把它升级为 `reports/` 风格

### 2) 逻辑抽离的边界

把以下内容从 notebook 抽到 `lib/` / `scripts/`：

- 会在多个 notebook 复用的函数/类
- 超过几十行、包含多分支/状态的逻辑
- 需要测试/重跑的逻辑（数据清洗、特征工程、聚合、评估）
- 任何你不想在 JSON diff 里审的东西

把以下内容留在 notebook：

- 解释性文字、假设、结论
- 少量 glue code（调用 `lib/`，组织步骤）
- 轻量的可视化与结果展示（但输出应可控）

### 3) 路径与项目根（Jupyter / Colab 都能跑）

notebook 环境下通常没有 `__file__`，所以“相对路径”依赖于当前工作目录。为了跨环境一致，推荐在第一段 Setup cell 里做两件事：

1) 自动定位项目根（例如找到 `pyproject.toml` 或 `.git`）
2) `chdir` 到项目根，并把项目根加入 `sys.path`，确保 `import lib...` 稳定

```python
from __future__ import annotations

from pathlib import Path
import os
import sys


def find_project_root(start: Path | None = None) -> Path:
    current = (start or Path.cwd()).resolve()
    for parent in [current, *current.parents]:
        if (parent / "pyproject.toml").exists() or (parent / ".git").exists():
            return parent
    raise RuntimeError("Project root not found (no pyproject.toml or .git found)")


PROJECT_ROOT = find_project_root()
os.chdir(PROJECT_ROOT)
sys.path.insert(0, str(PROJECT_ROOT))
```

对 Colab 的额外建议：

- 默认工作目录通常是 `/content`，建议把数据路径集中在一个变量里（例如 `DATA_DIR = PROJECT_ROOT / "data"`）
- 如果你必须挂载 Google Drive，把“挂载 + 路径选择”写成一个可选 cell，避免污染本地/Jupyter 的运行

### 4) Git 友好（ignore / lockfile / 输出）

- `data/` 与 `reports/` 默认 gitignored，但保留 `.gitkeep`
- `uv.lock`（或其它 lockfile）建议提交，保证环境可复现
- notebook 输出默认清理（见 `references/token-efficiency.md`），避免 diff 被大输出淹没

`.gitignore` 最核心的约定是：忽略数据与产物，保留结构与代码。示例可参考本 skill 的 `SKILL.md`。

## 迁移清单（把“杂乱 notebook 仓库”整理成工程）

1. 先确定“主入口 notebook”与“交付 notebook”（通常放根目录或 `notebooks/00-overview.ipynb`）
2. 创建 `lib/` 与 `scripts/`，把重复逻辑迁移进去
3. 创建 `data/raw`、`data/processed`、`reports/` 并补齐 `.gitignore` 与 `.gitkeep`
4. notebook 中只保留调用 `lib/` 的 glue code，并确保从零开始可跑通
5. 加上输出清理策略（pre-commit / nbstripout / nbconvert），让 diff 变干净
