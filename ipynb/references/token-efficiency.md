# Token 高效处理（.ipynb）

`.ipynb` 本质是 JSON，最大的“性能/可维护性”杀手通常不是代码，而是输出：大表格、base64 图片、HTML、日志等会让文件膨胀、diff 难审、AI 上下文被吞噬。

本文件提供一套“默认省 token”策略：先查询结构与代码，再按需读取；输出尽量落盘，notebook 保持可复现。

## 默认策略（强烈推荐）

1. **提交前清理输出**：notebook 只保留源码与必要 metadata
2. **审阅先看结构**：先统计 cell 类型与输出数量，再决定读哪里
3. **对比只看代码**：把 code cell 抽出来对比，避免 diff 被输出淹没
4. **大输出落盘**：图表/表格/模型产物存到 `reports/`，notebook 只打印摘要与路径

## 快速查询（不“阅读 notebook”，只做结构化扫描）

### 1) Cell 类型与数量

```bash
jq '.cells | group_by(.cell_type) | map({type: .[0].cell_type, count: length})' notebook.ipynb
```

### 2) 有输出的 code cell 数量

```bash
jq '[.cells[] | select(.cell_type == "code") | select(.outputs | length > 0)] | length' notebook.ipynb
```

### 3) 快速提取 Markdown 标题（理解叙事结构）

```bash
jq -r '.cells[] | select(.cell_type == "markdown") | .source[]? | select(startswith("#"))' notebook.ipynb
```

### 4) 抽出 code cell（用于对比/复审）

```bash
jq -r '.cells[] | select(.cell_type == "code") | "### CELL", (.source | join("")), ""' notebook.ipynb
```

### 5) 没有 jq 时的 Python 兜底

```bash
python - <<'PY'
from __future__ import annotations

import json
from pathlib import Path


def main() -> None:
    path = Path("notebook.ipynb")
    nb = json.loads(path.read_text(encoding="utf-8"))
    cells = nb.get("cells", [])
    counts: dict[str, int] = {}
    code_with_outputs = 0
    for cell in cells:
        cell_type = cell.get("cell_type", "unknown")
        counts[cell_type] = counts.get(cell_type, 0) + 1
        if cell_type == "code" and cell.get("outputs"):
            code_with_outputs += 1
    print({"counts": counts, "code_cells_with_outputs": code_with_outputs, "total_cells": len(cells)})


if __name__ == "__main__":
    main()
PY
```

## 清理输出（让 Git diff 与 AI 上下文变“可用”）

### 方案 A：pre-commit + nbstripout（推荐）

在仓库里配置 `nbstripout`，默认每次提交都会清理输出。示例见本 skill 的 `SKILL.md`。

### 方案 B：nbconvert 就地清理

```bash
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace notebook.ipynb
```

### 方案 C：批量清理（谨慎使用）

```bash
find . -name '*.ipynb' -print0 | xargs -0 -n 1 jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace
```

提示：批量清理前建议先 `git status`，确保不会误伤不该改动的 notebook。

## 对比策略（只对比你关心的部分）

### 1) 代码优先对比（最通用）

把 code cell 抽取到临时文件做 diff（见上面的 jq 示例）。

### 2) 专业 notebook diff（可选）

如果团队经常审 notebook diff，建议使用 `nbdime`：

```bash
uv add nbdime
uv run nbdiff notebook1.ipynb notebook2.ipynb
```

### 3) 文本化工作流（可选）

如果 notebook 需要长期维护且变更频繁，可以考虑 `jupytext` 生成配对的文本格式（例如 percent script），让核心逻辑在 `.py` 中可审。

## 编辑策略（尽量减少对 .ipynb JSON 的直接改动）

优先顺序建议：

1. **改 `lib/` 或 `scripts/`**，然后在 notebook 里重新运行
2. notebook 只做“调用与展示”，减少直接编辑的代码量
3. 需要程序化改 notebook 时，用 `nbformat` 读写（不要手工编辑 JSON）

```python
from __future__ import annotations

from pathlib import Path
import nbformat as nbf


path = Path("notebook.ipynb")
nb = nbf.read(path, as_version=4)

# Modify nb["cells"] here

nbf.write(nb, path)
```

## 输出管理（让 notebook 轻、可复现、可分享）

- 对大 DataFrame：输出 `shape`、列名、`head()`，避免全量展示
- 对图表：优先 `fig.write_html()` / `fig.write_image()` 落盘，并打印路径
- 对日志：避免把大量日志留在 output；必要时写文件或用采样/聚合后的摘要

如果你发现 notebook 文件体积异常增长，优先检查：

- code cell output 是否包含 base64（常见于 `fig.show()`、内联图片、HTML）
- 是否把大对象直接 `print()` / `display()`
- 是否把完整数据集/模型对象写进了 output
