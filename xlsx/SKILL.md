---
name: xlsx
description: "全面的电子表格创建、编辑和分析，支持公式、格式设置、数据分析和可视化。当需要处理电子表格（.xlsx, .xlsm, .csv, .tsv 等）以进行以下操作时使用：(1) 创建带有公式和格式的新电子表格，(2) 读取或分析数据，(3) 在保留公式的同时修改现有电子表格，(4) 电子表格中的数据分析和可视化，或 (5) 重新计算公式"
---

# 输出要求

## 所有 Excel 文件

### 零公式错误

  - 每个 Excel 模型交付时必须保证**零**公式错误（\#REF\!, \#DIV/0\!, \#VALUE\!, \#N/A, \#NAME?）

### 保留现有模板（当更新模板时）

  - 修改文件时，研究并**完全**匹配现有的格式、样式和惯例
  - 永远不要对已有既定模式的文件强制使用标准化格式
  - 现有模板的惯例**始终**优先于这些准则

## 财务模型

### 颜色编码标准

除非用户或现有模板另有说明

#### 行业标准颜色惯例

  - **蓝色文本 (RGB: 0,0,255)**：硬编码输入（Hardcoded inputs），以及用户为情景分析将更改的数字
  - **黑色文本 (RGB: 0,0,0)**：所有公式和计算
  - **绿色文本 (RGB: 0,128,0)**：从同一工作簿内的其他工作表引用的链接
  - **红色文本 (RGB: 255,0,0)**：指向其他文件的外部链接
  - **黄色背景 (RGB: 255,255,0)**：需要注意的关键假设或需要更新的单元格

### 数字格式标准

#### 必需的格式规则

  - **年份**：格式化为文本字符串（例如 "2024" 而不是 "2,024"）
  - **货币**：使用 $#,##0 格式；**务必**在标题中指定单位（"Revenue ($mm)"）
  - **零值**：使用数字格式将所有零显示为 "-"，包括百分比（例如 "$#,##0;($\#,\#\#0);-"）
  - **百分比**：默认为 0.0% 格式（保留一位小数）
  - **倍数**：估值倍数格式化为 0.0x（EV/EBITDA, P/E）
  - **负数**：使用括号 (123) 而不是负号 -123

### 公式构建规则

#### 假设放置

  - 将**所有**假设（增长率、利润率、倍数等）放在单独的假设单元格中
  - 在公式中使用单元格引用而不是硬编码值
  - 示例：使用 `=B5*(1+$B$6)` 而不是 `=B5*1.05`

#### 防止公式错误

  - 验证所有单元格引用是否正确
  - 检查范围内的差一错误（off-by-one errors）
  - 确保所有预测期间的公式一致
  - 测试边缘情况（零值、负数）
  - 验证没有非预期的循环引用

#### 硬编码的文档要求

  - 在单元格旁或注释中说明（如果是表格末尾）。格式："Source: [System/Document], [Date], [Specific Reference], [https://www.gymglish.com/en/gymglish/english-translation/if-applicable](https://www.gymglish.com/en/gymglish/english-translation/if-applicable)"
  - 示例：
      - "Source: Company 10-K, FY2024, Page 45, Revenue Note, [SEC EDGAR URL]"
      - "Source: Company 10-Q, Q2 2025, Exhibit 99.1, [SEC EDGAR URL]"
      - "Source: Bloomberg Terminal, 8/15/2025, AAPL US Equity"
      - "Source: FactSet, 8/20/2025, Consensus Estimates Screen"

# XLSX 创建、编辑和分析

## 概述

用户可能会要求你创建、编辑或分析 .xlsx 文件的内容。针对不同的任务，你有不同的工具和工作流可用。

## 重要要求

**重新计算公式需要 LibreOffice**：你可以假设已安装 LibreOffice，用于使用 `recalc.py` 脚本重新计算公式值。该脚本在首次运行时会自动配置 LibreOffice。

## 读取和分析数据

### 使用 pandas 进行数据分析

对于数据分析、可视化和基本操作，请使用 **pandas**，它提供了强大的数据处理能力：

```python
import pandas as pd

# 读取 Excel
df = pd.read_excel('file.xlsx')  # 默认：第一个工作表
all_sheets = pd.read_excel('file.xlsx', sheet_name=None)  # 所有工作表作为字典

# 分析
df.head()      # 预览数据
df.info()      # 列信息
df.describe()  # 统计信息

# 写入 Excel
df.to_excel('output.xlsx', index=False)
```

## Excel 文件工作流

## 关键：使用公式，而非硬编码值

**始终使用 Excel 公式，而不是在 Python 中计算值并硬编码它们。** 这确保了电子表格保持动态和可更新。

### ❌ 错误 - 硬编码计算值

```python
# 错误：在 Python 中计算并硬编码结果
total = df['Sales'].sum()
sheet['B10'] = total  # 硬编码了 5000

# 错误：在 Python 中计算增长率
growth = (df.iloc[-1]['Revenue'] - df.iloc[0]['Revenue']) / df.iloc[0]['Revenue']
sheet['C5'] = growth  # 硬编码了 0.15

# 错误：Python 计算平均值
avg = sum(values) / len(values)
sheet['D20'] = avg  # 硬编码了 42.5
```

### ✅ 正确 - 使用 Excel 公式

```python
# 正确：让 Excel 计算总和
sheet['B10'] = '=SUM(B2:B9)'

# 正确：作为 Excel 公式的增长率
sheet['C5'] = '=(C4-C2)/C2'

# 正确：使用 Excel 函数的平均值
sheet['D20'] = '=AVERAGE(D2:D19)'
```

这适用于**所有**计算——总计、百分比、比率、差值等。当源数据更改时，电子表格应能够重新计算。

## 常用工作流

1.  **选择工具**：pandas 用于数据，openpyxl 用于公式/格式
2.  **创建/加载**：创建新工作簿或加载现有文件
3.  **修改**：添加/编辑数据、公式和格式
4.  **保存**：写入文件
5.  **重新计算公式（如果使用公式则必须执行）**：使用 `recalc.py` 脚本
    ```bash
    python recalc.py output.xlsx
    ```
6.  **验证并修复任何错误**：
      - 脚本返回带有错误详细信息的 JSON
      - 如果 `status` 为 `errors_found`，请检查 `error_summary` 以了解具体的错误类型和位置
      - 修复识别出的错误并再次重新计算
      - 需要修复的常见错误：
          - `#REF!`：无效的单元格引用
          - `#DIV/0!`：除以零
          - `#VALUE!`：公式中的数据类型错误
          - `#NAME?`: 无法识别的公式名称

### 创建新的 Excel 文件

```python
# 使用 openpyxl 进行公式和格式设置
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment

wb = Workbook()
sheet = wb.active

# 添加数据
sheet['A1'] = 'Hello'
sheet['B1'] = 'World'
sheet.append(['Row', 'of', 'data'])

# 添加公式
sheet['B2'] = '=SUM(A1:A10)'

# 格式设置
sheet['A1'].font = Font(bold=True, color='FF0000')
sheet['A1'].fill = PatternFill('solid', start_color='FFFF00')
sheet['A1'].alignment = Alignment(horizontal='center')

# 列宽
sheet.column_dimensions['A'].width = 20

wb.save('output.xlsx')
```

### 编辑现有的 Excel 文件

```python
# 使用 openpyxl 保留公式和格式
from openpyxl import load_workbook

# 加载现有文件
wb = load_workbook('existing.xlsx')
sheet = wb.active  # 或 wb['SheetName'] 指定工作表

# 处理多个工作表
for sheet_name in wb.sheetnames:
    sheet = wb[sheet_name]
    print(f"Sheet: {sheet_name}")

# 修改单元格
sheet['A1'] = 'New Value'
sheet.insert_rows(2)  # 在位置 2 插入行
sheet.delete_cols(3)  # 删除第 3 列

# 添加新工作表
new_sheet = wb.create_sheet('NewSheet')
new_sheet['A1'] = 'Data'

wb.save('modified.xlsx')
```

## 重新计算公式

由 openpyxl 创建或修改的 Excel 文件包含作为字符串的公式，但不包含计算值。使用提供的 `recalc.py` 脚本重新计算公式：

```bash
python recalc.py <excel_file> [timeout_seconds]
```

示例：

```bash
python recalc.py output.xlsx 30
```

该脚本会：

  - 首次运行时自动设置 LibreOffice 宏
  - 重新计算所有工作表中的所有公式
  - 扫描**所有**单元格以查找 Excel 错误（\#REF\!, \#DIV/0\! 等）
  - 返回带有详细错误位置和计数的 JSON
  - 在 Linux 和 macOS 上均可运行

## 公式验证清单

确保公式正常工作的快速检查：

### 基本验证

  - [ ] **测试 2-3 个样本引用**：在构建完整模型之前验证它们是否提取了正确的值
  - [ ] **列映射**：确认 Excel 列匹配（例如，第 64 列 = BL，而不是 BK）
  - [ ] **行偏移**：记住 Excel 行是 1 索引的（DataFrame 行 5 = Excel 行 6）

### 常见陷阱

  - [ ] **NaN 处理**：使用 `pd.notna()` 检查空值
  - [ ] **极右列**：财年数据通常在 50 列之后
  - [ ] **多重匹配**：搜索所有出现的位置，而不仅仅是第一个
  - [ ] **除以零**：在公式中使用 `/` 之前检查分母（\#DIV/0\!）
  - [ ] **错误引用**：验证所有单元格引用是否指向预期单元格（\#REF\!）
  - [ ] **跨表引用**：使用正确的格式（Sheet1\!A1）链接工作表

### 公式测试策略

  - [ ] **从小处着手**：在广泛应用之前，先在 2-3 个单元格上测试公式
  - [ ] **验证依赖关系**：检查公式中引用的所有单元格是否存在
  - [ ] **测试边缘情况**：包括零、负数和非常大的值

### 解读 recalc.py 输出

脚本返回带有错误详细信息的 JSON：

```json
{
  "status": "success",          // 或 "errors_found"
  "total_errors": 0,              // 总错误计数
  "total_formulas": 42,           // 文件中的公式数量
  "error_summary": {              // 仅当发现错误时出现
    "#REF!": {
      "count": 2,
      "locations": ["Sheet1!B5", "Sheet1!C10"]
    }
  }
}
```

## 最佳实践

### 库选择

  - **pandas**：最适合数据分析、批量操作和简单数据导出
  - **openpyxl**：最适合复杂格式、公式和 Excel 特定功能

### 使用 openpyxl

  - 单元格索引基于 1（行=1，列=1 指的是单元格 A1）
  - 使用 `data_only=True` 读取计算值：`load_workbook('file.xlsx', data_only=True)`
  - **警告**：如果使用 `data_only=True` 打开并保存，公式将被值替换并永久丢失
  - 对于大文件：读取时使用 `read_only=True`，写入时使用 `write_only=True`
  - 公式会被保留但不会被求值 - 使用 recalc.py 更新值

### 使用 pandas

  - 指定数据类型以避免推断问题：`pd.read_excel('file.xlsx', dtype={'id': str})`
  - 对于大文件，读取特定列：`pd.read_excel('file.xlsx', usecols=['A', 'C', 'E'])`
  - 正确处理日期：`pd.read_excel('file.xlsx', parse_dates=['date_column'])`

## 代码风格指南

**重要**：生成用于 Excel 操作的 Python 代码时：

  - 编写最少、简洁的 Python 代码，没有不必要的注释
  - 避免冗长的变量名和多余的操作
  - 避免不必要的打印语句

**对于 Excel 文件本身**：

  - 为包含复杂公式或重要假设的单元格添加注释
  - 记录硬编码值的数据来源
  - 包含关键计算和模型部分的说明