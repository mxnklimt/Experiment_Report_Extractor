---
name: experiment-report-extractor
description: Use when user provides a directory of Markdown experiment logs (with optional data files) and wants to extract experiment name, results, and conclusions into a single self-contained HTML report with charts
---

# Experiment Report Extractor

## Overview

Scan a directory of Markdown experiment files and optional data files, extract three key fields per experiment, and render a modern card-layout HTML report with ECharts visualizations — all in a single self-contained HTML file.

## When to Use

- User provides a directory path with `.md` experiment reports
- User wants to generate an HTML report/summary from multiple experiment logs
- User mentions "汇报", "实验报告", "提取实验信息", "生成HTML"
- Data files (CSV, JSON, etc.) may be present alongside the markdown files

## Core Workflow

### Step 1 — Scan the Directory

Read all files in the directory:
- Collect all `.md` files → experiment reports
- Collect all data files (`.csv`, `.tsv`, `.json`, `.jsonl`, `.xlsx`) → potential chart sources

### Step 2 — Associate Data Files with Experiments

Priority order:
1. **Same-name match**: `exp1.md` + `exp1.csv` → auto-associated
2. **Directory grouping**: files inside `experiments/exp1/` → all data files belong to `exp1.md` in same dir
3. **Manual override**: if user explicitly says "exp1.md uses data.csv", respect that

### Step 3 — Extract Three Fields per Experiment

For each `.md` file, extract:
- **实验名称** (Experiment Name)
- **实验结果** (Results)
- **结论** (Conclusion)

**Extraction strategy — rule-first, LLM fallback:**

```
Rule Phase (try in order):
  实验名称:
    1. First `#` heading in file
    2. `## 实验名称` / `## 标题` / `## 实验` section
    3. Filename (strip extension, replace hyphens/underscores with spaces)

  实验结果:
    1. `## 结果` / `## 实验结果` / `## 数据` / `## 测试结果` section content
    2. Fuzzy: any heading matching /结果|数据|测试|观测/
    3. First markdown table found in the file

  结论:
    1. `## 结论` / `## 总结` / `## 讨论` / `## 下一步` section content
    2. Fuzzy: any heading matching /结论|总结|讨论|建议/
    3. Last paragraph of the document

LLM Fallback (if any field is still missing after rule phase):
  Prompt: "从以下Markdown文档中提取：1)实验名称 2)实验结果(核心发现) 3)结论。
  请用JSON格式返回：{name, results, conclusion}。
  如果某字段确实不存在，返回null。"
  
  Use the full markdown content as input.
```

If a field cannot be extracted even with LLM fallback, display **"未提取到该信息"** in the HTML card.

### Step 4 — Parse Data Files

For each associated data file:

| Format | Parsing approach |
|--------|-----------------|
| `.csv` / `.tsv` | Auto-detect delimiter, parse headers, convert numeric columns |
| `.json` | If array of objects → use keys as columns; if nested → flatten one level |
| `.jsonl` | Each line as a row |
| Other / ambiguous | Use LLM to interpret structure and return as `{headers: [], rows: [[]]}` |

**Chart type selection (auto):**

| Data shape | Chart type |
|-----------|------------|
| 2 columns: one index-like + one numeric | Line chart |
| 1 categorical column + 1 numeric | Bar chart |
| 3+ numeric columns | Multi-series line chart |
| 2 numeric columns (neither is clearly an index) | Scatter chart |
| User specifies `<!-- chart: bar -->` in the `.md` | Override with specified type |

### Step 5 — Generate Self-Contained HTML

**CRITICAL REQUIREMENTS:**
- ✅ Use **ECharts** for all charts (NOT Chart.js, NOT Plotly, NOT D3)
- ✅ Embed ECharts library **inline** in the HTML `<script>` tag (download/use local copy)
- ✅ Output a **single `.html` file** with zero external dependencies
- ✅ The file must work when opened offline in a browser

**Readability requirements (MANDATORY — do NOT make these smaller):**

| Element | Minimum spec |
|---------|-------------|
| Chart height | **360px minimum** (400px preferred) |
| Chart width | 100% of card width |
| Chart axis label font size | **14px** |
| Chart tooltip font size | **14px** |
| Chart legend font size | **14px** |
| Body / field value text | **15px** |
| Field label (uppercase tag) | **12px** |
| Card title | **18px** |
| Table cell font size | **14px** |
| Conclusion box font size | **15px** |

These values ensure the report is comfortably readable without zooming. Never reduce them for "compact" or "aesthetic" reasons.

**Anti-overlap rules (MANDATORY — text overlap is a hard error):**

X-axis labels:
- Always set `axisLabel.rotate` to **45** when there are more than 4 data points, or when any label is longer than 4 characters
- Set `axisLabel.interval: 0` to force all labels to show, then rely on rotation to avoid overlap
- Never let ECharts auto-skip labels (default `interval: 'auto'` hides labels — forbidden)

Data labels on bars/lines:
- If bars are narrow or values are close together, set `label.position: 'top'` and ensure chart height is tall enough that labels don't collide
- For dense bar charts (more than 6 bars), hide data labels (`label.show: false`) and rely on tooltip instead
- Never use `label.position: 'inside'` if the bar is too short to contain the text

Legend:
- If legend items are many, set `legend.type: 'scroll'` to prevent overflow
- Set `legend.padding: [5, 20]` and `legend.itemGap: 16` to give items breathing room

Grid margins — always reserve enough space for labels:
```js
grid: {
  top: 40,       // space for legend/title
  right: 30,     // space for last x-axis label
  bottom: 70,    // space for rotated x-axis labels (use 100 if labels are long)
  left: 70       // space for y-axis labels and name
}
```

General rule: **when in doubt, increase grid margins and rotate labels.** Never sacrifice readability for a tighter layout.

**HTML structure:**
```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="UTF-8">
  <title>实验汇报</title>
  <style>
    /* All CSS inline here */
    /* Color palette: #f5f6fa background, #fff cards, #4a90d9 accent */
    /* Font sizes per readability table above */
  </style>
</head>
<body>
  <!-- Summary bar: N experiments, M charts -->

  <!-- Card grid: one card per experiment -->
  <!-- Each card:
    - Header: experiment name (18px) + source filename
    - Results section: text (15px) + optional table (14px)
    - Conclusion section: distinct background color, 15px font
    - ECharts chart div: min-height 360px (if data file associated)
  -->

  <script>
    /* ECharts full library source code inline here */
    /* All chart initialization code here */
    /* ECharts textStyle.fontSize: 14 for all axis/legend/tooltip */
  </script>
</body>
</html>
```

**How to get ECharts inline:**
1. Read ECharts from a local installation if available: `node_modules/echarts/dist/echarts.min.js`
2. Or use the Bash tool to download it: `curl -s https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js`
3. Embed the full JS source as a `<script>` block

**Output file:** Save as `report.html` in the same directory as the input files, or where the user specifies.

## Quick Reference

```
Input:  directory path (+ optional manual file associations)
Output: report.html (self-contained, offline-ready)

Fields: 实验名称 | 实验结果 | 结论
Charts: ECharts inline, auto-type selection
Fallback: LLM extraction when rules fail
Missing: "未提取到该信息"
```

## Common Mistakes

| Mistake | Correct behavior |
|---------|-----------------|
| Using Chart.js or Plotly | **Always use ECharts** |
| Linking to CDN for ECharts | **Inline the full ECharts source** in `<script>` |
| Adding extra fields beyond the 3 specified | Only extract: 实验名称, 实验结果, 结论 |
| Inventing/calculating data not in source files | Only use data present in the files |
| Saving to `/tmp/` or system paths | Save `report.html` to the user's input directory |
| Skipping LLM fallback for free-format markdown | Always attempt LLM fallback when rules fail |
| Making charts too small (height < 300px) | Chart height ≥ 360px, all fonts ≥ 14px |
| Using small font sizes for "clean" look | Body text 15px, axis/table/tooltip 14px minimum |
| X-axis labels overlapping each other | Set `axisLabel.rotate: 45` + `interval: 0` when > 4 points or labels > 4 chars |
| Data labels on bars overlapping | Hide labels (`label.show: false`) for dense charts (> 6 bars); use tooltip instead |
| Grid too tight, labels cut off | Use `grid: {bottom: 70, left: 70, right: 30, top: 40}` as minimum margins |

## Example Invocation

User: "帮我处理 /home/user/experiments/ 目录，生成实验汇报"

You:
1. List files in `/home/user/experiments/`
2. For each `.md`: run extraction (rules → LLM fallback)
3. For each data file: parse + select chart type
4. Fetch/embed ECharts source
5. Render HTML with all cards
6. Write to `/home/user/experiments/report.html`
7. Confirm: "已生成 report.html，包含 N 个实验卡片，M 个图表"
