# 演示 / 分享模式（.ipynb）

本文件给出一套“可分享、可复现、可导出”的 notebook 模式：让读者打开 notebook 就能快速理解背景、复现结果，并且不会被输出噪音淹没。

## 目标

- 打开即懂：标题、背景、数据来源与结论清晰
- 可复现：从干净环境 “Restart & Run All” 能跑通
- 可审阅：变更主要体现在代码与叙事，不被输出污染
- 可导出：能稳定导出 HTML（必要时导出执行版 notebook）

## 推荐结构模板

你可以把 notebook 组织成固定分段（每段用 Markdown 标题标记）：

1. 标题与结论摘要（1 屏内讲清楚）
2. 环境与配置（版本、随机种子、路径、参数）
3. 数据加载（输入路径、校验、失败时提示）
4. 指标/统计摘要（只输出关键数字与小表）
5. 可视化与解释（图 + 结论句）
6. 结论与下一步（明确假设、限制与待做）

## 可复现：把“环境差异”收敛到一处

### 1) 统一配置区（单独一个 cell）

- 参数（日期范围、阈值、采样率等）集中在一个 cell
- 统一随机种子
- `PROJECT_ROOT`、`DATA_DIR`、`REPORT_DIR` 等路径变量集中定义

### 2) Colab 适配（可选 cell，避免污染本地/Jupyter）

如果你需要在 Colab 中运行，常见做法是：

- 用 `%pip install` 安装依赖（建议加 `-q` 控噪音）
- 可选地挂载 Google Drive（如果数据在 Drive）

```python
%pip install -q pandas plotly
```

```python
from google.colab import drive

drive.mount("/content/drive")
```

注意：上面的 cell 建议明确标记为“Colab only”，并且不要让本地/Jupyter 必须依赖它才能运行。

## 输出规范：少而精，可追溯

### 1) 输出摘要而不是全量

- DataFrame：优先 `df.shape`、列名、`df.head()`，必要时加分组聚合后的 Top-N
- 日志：避免输出大量日志到 cell output

### 2) 大输出落盘并打印路径

```python
from __future__ import annotations

from pathlib import Path
from datetime import datetime


today = datetime.now().strftime("%Y-%m-%d")
report_dir = Path("reports") / today
report_dir.mkdir(parents=True, exist_ok=True)

fig.write_html(report_dir / "chart.html")
print(f"[OK] Saved: {report_dir}/chart.html")
```

如果你在 Windows 环境不方便用 symlink，可以用一个 `reports/LATEST.txt` 记录最新目录名，或让 README 指向最新报告路径。

## 导出与自动执行

### 1) 导出 HTML（最通用）

```bash
jupyter nbconvert --to html --execute notebook.ipynb
```

### 2) 产出“执行后的 notebook”（用于归档）

```bash
jupyter nbconvert --to notebook --execute --inplace notebook.ipynb
```

### 3) PDF（可选）

PDF 通常依赖 LaTeX 环境，团队如果没有统一环境，建议优先 HTML。

## 分享前检查清单

1. “Restart & Run All” 跑通（本地/Jupyter 与 Colab 至少有一个环境能完整跑通）
2. 输出被控制（无大表格、无大量日志、无 base64 大块）
3. 秘密信息清理（token、cookie、账号、内网 URL 等）
4. `reports/` 产物路径正确，读者按说明能找到导出的结果
5. 提交前清理输出（配合 `nbstripout` / `nbconvert --clear-output`）
