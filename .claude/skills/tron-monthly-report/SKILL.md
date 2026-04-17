---
name: tron-report
description: >
  TRON 月报/季报自动生成。从 Google Sheet 或本地 Excel 读取数据，修改脚本配置，运行脚本，输出 Word 文档并上传至 Google Doc。
  触发词：月报、季报、生成报告、数据更新了、月报脚本、tron report、generate monthly report、generate quarterly report、
  4月月报、Q1季报、帮我跑月报、帮我跑季报、月报、季报、数据更新了。
---

# TRON 月报/季报自动生成

## 脚本位置

| 报告类型 | 脚本路径 |
|---------|---------|
| 月报 | `/Users/admin/data-analysis-tron/generate_monthly_report.py` |
| 季报 | `/Users/admin/data-analysis-tron/generate_report.py` |

---

## 月报流程

### Step 1：修改配置区

编辑脚本中的配置区（约第33-34行）：

```python
CUR_MONTH  = "YYYY-MM"   # 当期月
PREV_MONTH = "YYYY-MM"   # 对比月（通常是上个月）
```

### Step 2：运行脚本

```bash
cd /Users/admin/data-analysis-tron
python3 generate_monthly_report.py
```

### Step 3：确认输出

- 本地副本：`/Users/admin/Downloads/{CUR_MONTH}月报_debug.docx`
- Google Doc：`https://docs.google.com/document/d/1BIevmdjQM2BQD3RJDixne7kdnhfFtz1HVsxa9rDCBfY/edit`

---

## 季报流程

### Step 1：修改配置区

编辑脚本中的配置区（约第18-30行）：

```python
OUTPUT_PATH = "/Users/admin/Downloads/YYYY-Qx 季报.docx"

Q1_MONTHS  = ['YYYY-01', 'YYYY-02', 'YYYY-03']  # 当期季度各月（按实际季度改）
Q4_MONTHS  = ['YYYY-10', 'YYYY-11', 'YYYY-12']  # 对比季度各月

Q1_DAYS    = 90   # 当期季度总天数（需手动计算）
Q4_DAYS    = 92   # 对比季度总天数（需手动计算）

MONTH_DAYS = {    # 更新各月天数
    'YYYY-MM': 31, ...
}
```

常用季度天数：Q1(非闰年)=90, Q1(闰年)=91, Q2=91, Q3=92, Q4=92

### Step 2：运行脚本

```bash
cd /Users/admin/data-analysis-tron
python3 generate_report.py
```

### Step 3：确认输出

- 本地文件：OUTPUT_PATH 配置的路径

---

## 数据源说明

详见 [references/data_sources.md](references/data_sources.md)

---

## 关键注意事项

- 月报数据源是 Google Sheet（自动拉取），季报数据源是本地 Excel（需提前更新）
- 若输出中出现大量 `N/A`，说明某个 sheet 缺数据，按报错提示定位
- 月报脚本依赖 `gsheet-confluence/read_google_sheet.py`，需确保该文件存在
- 季报天数需手动计算，不会自动推算（月报已自动计算）
