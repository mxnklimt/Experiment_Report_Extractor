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
- User specifies exactly two `.md` files: a **benchmark report** + a **code analysis** file — skip directory scanning and use **Two-File Mode**

## Two-File Mode: Benchmark Report + Code Analysis

When the user explicitly specifies exactly two `.md` files — typically a **benchmark report** (containing performance comparison data) and a **code analysis** (explaining optimization techniques) — skip the standard directory-scanning workflow and process these two files together into a unified report.

### Identifying the Two Files

If the user doesn't explicitly label which is which, infer from content:

| Clue | Benchmark file | Analysis file |
|------|---------------|---------------|
| Keywords | `QPS`, `P50`/`P90`/`P99`, `延迟`, `压测`, `bench`, `benchmark` | `代码对比`, `code`, `优化`, `for-loop`, `向量化`, `analysis`, `源码` |
| Structure | Multiple data tables with numeric columns, comparison groups (A/B/C) | Code blocks (` ```python `), optimization breakdown tables, line-by-line diffs |
| Typical name | `BENCHMARK_REPORT.md`, `bench_results.md` | `*_analysis.md`, `*_optimization.md` |

### Extracting Benchmark Data

From the benchmark file, extract these structured sections:

1. **Experiment Name**: First `#` heading in the file
2. **Experiment Config**: Table under `## 实验配置` or `## 配置`
3. **Comparison Groups**: Identify all named groups (A, B, C, etc.) from markdown tables. Each group has a unique name, script file reference, and method description.
4. **Detailed Results per Group**: For each group (A/B/C), extract the **average row** (typically the bolded `**avg**` row) from its results table. Fields include: QPS, Avg latency, P50, P90, P95, P99, and optional breakdown timings (Redis, Concat, Infer, etc.).
5. **Comparison Pair Tables**: Extract A vs B, B vs C, C vs A tables — these contain relative deltas (△).
6. **Results/Conclusion**: Content under headings matching `/对比分析|结论|总结|耗时分解/`.

**Data extraction for charts**: The average rows from each group's detailed results table are the primary chart data source. Parse these rows into structured arrays for ECharts:

```
Groups: [A, B, C]
QPS:    [27.69, 27.57, 29.21]
Avg:    [143.9, 144.5, 136.3]
Concat: [15.1, 15.0, 3.4]
Infer:  [115.9, 116.8, 121.7]
```

### Extracting Code Analysis

From the analysis file, preserve the full structure without data loss:

1. **Analysis Title**: First `#` heading
2. **Overview/Summary**: All content between the title and the first `---` horizontal rule (or first `##` section)
3. **Section-by-section body**: Each `##` section as a distinct block with its heading, explanatory text, comparison tables, and code blocks
4. **Code Blocks**: ALL ` ```python ` (and other language) fenced code blocks — preserve **exact** formatting, line breaks, and indentation. Never truncate or summarize code.
5. **Optimization Breakdown Table**: Table under headings like `## 总体加速分解` or `## 加速分解` — maps each optimization technique to its estimated gain
6. **Core Principles / Summary**: The final heading section (e.g., `## 核心原则总结`) that synthesizes the findings

### Rendering the Two-File Report

The HTML report for two-file mode has a **main body + appendix layout**:

**Main Body:**

**Card 1 — Benchmark Results** (from benchmark file):
- Experiment name as card title
- Experiment config as a compact summary table
- Comparison group overview table (group name, script, method)
- ECharts bar charts for key metrics — **one chart per metric**, each comparing all groups:
  - **Chart 1**: QPS comparison (one bar per group)
  - **Chart 2**: Avg latency comparison (one bar per group)
  - **Chart 3**: P50 latency comparison (one bar per group)
  - **Chart 4**: P90/P95/P99 tail latency comparison (if data available, one metric per chart)
  - **Chart 5+**: Time breakdown components (Redis / Concat / Infer, one chart per component, or one chart with carefully chosen dual Y-axis for components of vastly different scale)
- Per-metric charts are arranged in a 2-3 column grid (`chart-row` or `chart-row-3`), each chart with `min-height: 380px`
- Comparison pair tables (A vs B, B vs C, C vs A) rendered as HTML tables
- Conclusion text

**Card 2 — Combined Summary**:
- Synthesize benchmark conclusions with code analysis findings
- Key takeaway: quantitative result (e.g., "QPS +5.5%, concat 4.4x faster") + root cause (which code change enabled it)
- Brief mention of the optimization approach at a high level (1-2 sentences), but **do NOT include detailed code blocks or line-by-line analysis here**

**Appendix — 代码详细分析** (at the very end of the report):

**CRITICAL: ALL detailed code analysis content MUST be placed in the appendix, NOT in the main body cards.** The appendix is the last section of the report, after all main body cards. The only exception is when the user explicitly requests code analysis to appear inline in the main body.

The appendix contains the full code analysis from the analysis file:
- Analysis title (as appendix heading)
- Overview/summary
- Each `##` section rendered as a subsection with heading
- Code comparison tables (old vs new) rendered as HTML tables
- All ` ```python ` code blocks rendered with dark background, monospace font, syntax preserved exactly
- Optimization breakdown table rendered as HTML table
- Core principles / final summary

**Rationale:** Detailed code analysis (especially multi-line code blocks and line-by-line diffs) is reference material. Placing it in the main body interrupts the flow from data → charts → conclusion. The appendix keeps it accessible while keeping the main report focused on results and insights. Readers who want the implementation details can scroll to the appendix; readers who only need the conclusions can stop after Card 2.

### Code Block Rendering (for analysis file)

Apply this CSS for all code blocks from the analysis file:

```css
.code-block {
  background: #1e1e1e;
  color: #d4d4d4;
  padding: 16px 20px;
  border-radius: 8px;
  overflow-x: auto;
  font-family: 'Consolas', 'Courier New', monospace;
  font-size: 13px;
  line-height: 1.6;
  margin: 12px 0;
  white-space: pre;
  max-width: 100%;
  border: 1px solid #333;
}
.code-inline {
  background: #f0f0f0;
  color: #c7254e;
  padding: 2px 6px;
  border-radius: 3px;
  font-family: 'Consolas', 'Courier New', monospace;
  font-size: 0.9em;
}
```

### Two-File Mode Quick Reference

```
Input:  benchmark.md + analysis.md (two specific files, not a directory)
Output: report.html (self-contained, offline-ready)

Cards:   1. Benchmark Results (tables + ECharts bar charts)
         2. Code Analysis (formatted text + code blocks)
         3. Combined Summary (synthesized conclusion)

Charts:  QPS bar chart + Latency grouped bar + Time breakdown bar
Data:    Extracted from benchmark comparison group average rows
Code:    All code blocks from analysis file preserved exactly
```

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
| Chart height | **400px minimum** (480px preferred) |
| Chart width | 100% of card width |
| Chart axis label font size | **16px** |
| Chart tooltip font size | **16px** |
| Chart legend font size | **16px** |
| Body / field value text | **17px** |
| Field label (uppercase tag) | **14px** |
| Card title | **20px** |
| Table cell font size | **17px** |
| Conclusion box font size | **17px** |

These values ensure the report is comfortably readable without zooming. Never reduce them for "compact" or "aesthetic" reasons.

**Anti-overlap rules (MANDATORY — text overlap is a hard error):**

X-axis labels (MUST be horizontal — NEVER rotate or slant):
- X-axis labels must be displayed **horizontally** (`axisLabel.rotate: 0`, or omit the property entirely). Rotated, slanted, or diagonal text is **forbidden** — it is uncomfortable to read and looks unprofessional.
- When there are more than **6** X-axis categories, **split the data into multiple smaller charts** instead of crowding labels. Each chart should have ≤ 6 categories.
- **Abbreviate each label to 6 characters or fewer** wherever possible (e.g. "A: 内联for循环" → "A:内联for"). The full name can appear in the tooltip or legend.
- For labels that genuinely cannot be abbreviated, set `axisLabel.overflow: 'truncate'` and `axisLabel.width: 80` to show an ellipsis rather than letting text collide.
- Set `axisLabel.interval: 0` to force all labels to show (never let ECharts auto-skip labels with `interval: 'auto'`).
- When a chart has 5+ categories, it **must** be placed in a full-width row (not inside a `chart-row` or `chart-row-3` grid column) to give labels enough horizontal room.

Data labels on bars/lines:
- If bars are narrow or values are close together, set `label.position: 'top'` and ensure chart height is tall enough that labels don't collide
- For dense bar charts (more than 6 bars), hide data labels (`label.show: false`) and rely on tooltip instead
- Never use `label.position: 'inside'` if the bar is too short to contain the text

**Per-metric comparison (MANDATORY for benchmark/对比 reports):**

When comparing multiple groups across multiple metrics, the correct approach is **one chart per metric**, NOT one chart with many metric series:

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| 1 chart with 5 series (Avg/P50/P90/P95/P99) × 3 groups | 5 separate charts, each comparing 3 groups on one metric |
| User cannot visually compare A vs B vs C for P50 | P50 chart: 3 bars (A/B/C), differences immediately visible |
| Series legend distracts from group comparison | No legend needed — X-axis labels ARE the groups |

**Rule: each ECharts chart answers exactly ONE comparison question.** "Which group has the lowest P50 latency?" is one question → one chart. Bundling P50, P90, P95, P99 into one chart tries to answer 4 questions at once and answers none of them well.

When splitting a multi-metric chart into per-metric charts:
- Each chart's X-axis = the comparison groups (A, B, C), ≤ 6 categories
- Each chart's Y-axis = the single metric being compared
- Each chart's title = the metric name (e.g. "P50 延迟对比")
- Use small multiples layout: 2-3 charts per row in a `chart-row` grid, each with `min-height: 380px`
- For ≤ 3 groups, show data labels on bars (`label.show: true, position: 'top'`)
- Apply the Y-axis baseline rule independently to each chart (non-zero if diff < 10%)

Example for QPS + Avg + P50 + P99: produce 4 separate bar charts, each with 3 bars (A/B/C). The reader instantly sees which group wins on each metric without deciphering a legend.

Y-axis baseline for bar charts (non-zero start for close comparisons):
- **When the relative difference between the max and min bar values is less than 10%** (i.e. `(max - min) / max < 0.10`), the Y-axis **must NOT start from 0**. Starting from 0 would flatten the bars and make the comparison invisible.
- Set `yAxis.min` to approximately `min_value × 0.85` (rounded to a clean tick value). This crops the Y-axis to the data range so small differences become visually apparent.
- **Always** set `yAxis.min` explicitly — never rely on ECharts auto-scale, which may or may not start from 0.
- For bar charts with differences ≥ 10%, keep `yAxis.min: 0` (or omit `min`) — the standard 0-baseline preserves honest visual proportion.
- Example: bars at 27.57 and 29.21 (diff = 5.6%) → `yAxis.min: 23` (≈ 27.57 × 0.85, rounded). Bars at 100 and 200 (diff = 50%) → keep 0-baseline.
- **Always add a subtitle or annotation** when using non-zero baseline: `<div style="font-size:12px;color:#95a5a6;text-align:center;margin-top:-8px;">注：Y轴起点为 {min_value}，以突出微小差异</div>` below the chart container.

Legend:
- If legend items are many, set `legend.type: 'scroll'` to prevent overflow
- Set `legend.padding: [5, 20]` and `legend.itemGap: 16` to give items breathing room

Grid margins — always reserve enough space for labels:
```js
grid: {
  top: 55,       // space for legend/title (use 75 if legend has many items)
  right: 45,     // space for data labels on rightmost bar/point
  bottom: 65,    // space for horizontal x-axis labels
  left: 80       // space for y-axis labels and name
}
```

General rule: **when in doubt, increase grid margins, abbreviate labels, or split into multiple charts.** Never sacrifice readability for a tighter layout, and never rotate/slant text as a workaround for crowding.

**HTML content overlap prevention (MANDATORY):**

These rules apply to the HTML/CSS outside of ECharts — cards, tables, text sections, and code blocks.

Table cells:
- Always set `table-layout: fixed` on `<table>` and assign percentage widths to `<th>` columns. This prevents browsers from auto-collapsing narrow columns and overlapping adjacent cells.
- Add `word-break: break-all` or `overflow-wrap: break-word` to `<td>` to prevent long unbroken strings (URLs, paths, numbers) from overflowing cell boundaries.
- Set a minimum column width of 60px for narrow data columns to prevent crunching.
- When a table has text-heavy columns, set `td { vertical-align: top; }` so multi-line text doesn't vertically misalign and appear to overlap.

Card body spacing:
- Set `padding: 20px 30px` minimum on card bodies so text never touches card edges.
- Add `margin-bottom: 18px` between consecutive sections (tables, charts, text blocks) within a card.
- Code blocks inside cards: set `overflow-x: auto` and `max-width: 100%` so long lines scroll instead of overflowing the card.

Chart container spacing:
- Each `.chart-container` div must have `margin: 16px 0` minimum to separate it from adjacent content.
- Never place two charts in the same `.chart-container` div — each chart gets its own container.
- When placing charts side-by-side with `chart-row` (2 columns) or `chart-row-3` (3 columns), ensure each container has `min-height: 380px` so charts don't collapse vertically.

Title and heading spacing:
- Card titles (`18px`) must have `margin-bottom: 4px` for the source filename line below.
- Section labels (`12px uppercase`) must have `margin-bottom: 8px` before their corresponding content.
- Consecutive headings without content between them indicate a layout problem — add spacing or merge them.

Text content:
- Always set `line-height: 1.7` on body text (15px) to prevent line-to-line overlap in multi-line content.
- For `.field-value` sections with long pre-formatted text, add `white-space: pre-wrap` and `overflow-wrap: break-word`.
- Never set `line-height: 1` or `line-height: 1.2` on content text — tight line heights cause ascender/descender collision in Chinese + Latin mixed text.

**HTML structure:**
```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="UTF-8">
  <title>实验汇报</title>
  <style>
    /* All CSS inline here */
    /* Color palette: #f0f2f5 background, #fff cards, #4a90d9 accent */
    /* Font sizes per readability table above */

    /* MANDATORY anti-overlap rules: */
    /* - body: line-height >= 1.7, font-size 17px */
    /* - table: table-layout: fixed, word-break: break-word */
    /* - td, th: padding >= 9px 12px, vertical-align: top, font-size 17px */
    /* - .card-body: padding >= 20px 30px */
    /* - .chart-container: margin >= 16px 0, min-height >= 400px */
    /* - code blocks: overflow-x: auto, max-width: 100% */
    /* - .field-value: white-space: pre-wrap, overflow-wrap: break-word */
    /* - Never use line-height < 1.5 on any content text */
    /* - Card titles: font-size 20px, margin-bottom 4px */
  </style>
</head>
<body>
  <!-- Summary bar: N experiments, M charts -->

  <!-- Card grid: one card per experiment -->
  <!-- Each card:
    - Header: experiment name (20px) + source filename
    - Results section: text (17px) + optional table (16px)
    - Conclusion section: distinct background color, 17px font
    - ECharts chart div: min-height 400px (if data file associated)
  -->

  <script>
    /* ECharts full library source code inline here */
    /* All chart initialization code here */
    /* ECharts textStyle.fontSize: 16 for all axis/legend/tooltip */
  </script>
</body>
</html>
```

**How to get ECharts inline (CRITICAL — do NOT use PowerShell string replacement):**

PowerShell's string interpolation (`$var` expansion) and regex-based `-replace` will CORRUPT the ECharts JavaScript source (~1MB of minified code contains `$` characters that trigger dollar-variable expansion and regex syntax errors). **Never use PowerShell to splice ECharts into HTML.**

Reliable approach — two-phase assembly:

**Phase A: Write the HTML skeleton with a placeholder**
Use the **Write** tool to write the full HTML body (CSS, cards, tables, chart `<div>`s, and all chart init JS). Where the ECharts library itself belongs, put a unique placeholder token:

```
__ECHARTS_LIBRARY_GOES_HERE__
```

Do NOT include any ECharts library code in this Write call — just the placeholder string.

**Phase B: Replace the placeholder with the real ECharts source**
Choose one of these reliable methods:

| Method | Command |
|--------|---------|
| **Node.js (preferred)** | Write a one-shot script that reads both files and splices them with `String.replace()`. JavaScript treats all content as opaque strings — no interpolation, no regex syntax clashes. |
| **Python** | Same pattern: `html.replace('__ECHARTS_LIBRARY_GOES_HERE__', echarts_js)` |
| **Bash** | `sed` with a unique delimiter — but test on a small file first |

Node.js template (save to `_splice.js`, run with `node _splice.js`, then remove):
```js
const fs = require('fs');
const html = fs.readFileSync('report.html', 'utf8');
const echarts = fs.readFileSync('echarts.min.js', 'utf8');
fs.writeFileSync('report.html', html.replace('__ECHARTS_LIBRARY_GOES_HERE__', echarts), 'utf8');
```

**Phase C: Verify and clean up**
- Confirm the output file has exactly **2** `<script>` tags (one ECharts library, one chart init code)
- Confirm the placeholder token **NO LONGER** appears in the output
- Delete the intermediate files (`_splice.js`, `echarts.min.js`)

**Output file:** Save as `report.html` in the same directory as the input files, or where the user specifies.

## Quick Reference

**Standard Mode (directory scan):**
```
Input:  directory path (+ optional manual file associations)
Output: report.html (self-contained, offline-ready)

Fields: 实验名称 | 实验结果 | 结论
Charts: ECharts inline, auto-type selection
Fallback: LLM extraction when rules fail
Missing: "未提取到该信息"
```

**Two-File Mode (benchmark + analysis):**
```
Input:  benchmark.md + analysis.md (two specific files)
Output: report.html (self-contained, offline-ready)

Cards:   1. Benchmark Results (config + tables + per-metric ECharts bar charts)
         2. Combined Summary (synthesized conclusion + brief optimization mention)
         附录. 代码详细分析 (full code blocks, diffs, breakdown tables — at end of report)

Charts:  One chart per metric (QPS chart, Avg chart, P50 chart, P99 chart, etc.), each comparing all groups with 1 bar per group
Data:    Parsed from comparison group average rows in benchmark
Code:    All code blocks from analysis file preserved exactly, placed in appendix
```

## Common Mistakes

| Mistake | Correct behavior |
|---------|-----------------|
| Using Chart.js or Plotly | **Always use ECharts** |
| Linking to CDN for ECharts | **Inline the full ECharts source** in `<script>` |
| Adding extra fields beyond the 3 specified | **Standard mode only**: extract 实验名称, 实验结果, 结论. **Two-file mode**: extract benchmark data (groups, metrics, comparisons) + analysis (code blocks, breakdown tables, principles) |
| Inventing/calculating data not in source files | Only use data present in the files |
| Saving to `/tmp/` or system paths | Save `report.html` to the user's input directory |
| Skipping LLM fallback for free-format markdown | Always attempt LLM fallback when rules fail |
| Making charts too small (height < 400px) | Chart height ≥ 400px, all chart fonts ≥ 16px |
| Using small font sizes for "clean" look | Body text 17px, axis/table/tooltip 16px minimum |
| X-axis labels overlapping each other | Abbreviate labels to ≤ 6 chars; split into multiple charts when > 6 categories. **NEVER rotate/slant labels — horizontal text only.** |
| Data labels on bars overlapping | Hide labels (`label.show: false`) for dense charts (> 6 bars); use tooltip instead. For ≤ 6 bars, place labels above bars with adequate chart height. |
| Y-axis starts at 0 when bars differ by < 10% | Calculate `(max-min)/max`; if < 0.10, set `yAxis.min` to `min×0.85` (rounded). Add annotation: "注：Y轴起点为{min}，以突出微小差异". |
| Grid too tight, labels cut off | Use `grid: {bottom: 65, left: 80, right: 45, top: 55}` as minimum margins. Increase when labels are long. |
| Rotated or diagonal X-axis text | **Rotated text is forbidden.** Abbreviate labels or split charts to keep text horizontal. |
| Table cells with text overflowing or overlapping | Use `table-layout: fixed` with percentage column widths; add `word-break: break-word` to `<td>`; set `vertical-align: top`. |
| Chart containers too close together (charts touch) | Each `.chart-container` must have `margin: 16px 0` minimum; never put two charts in the same container div. |
| Tight line-height causing text overlap | Body text `line-height: 1.7` minimum; never use `line-height: 1` or `1.2` on content. |
| Using PowerShell `-replace` or here-string to splice ECharts inline | Use **Node.js or Python** for the placeholder-replace step. See "How to get ECharts inline" above for the exact two-phase process. PowerShell dollar-variable expansion CORRUPTS JavaScript containing `$` characters. |
| Writing the entire 1MB+ HTML in one tool call (Write/heredoc hangs or corrupts) | Split work into Phase A (Write HTML skeleton with placeholder), Phase B (Node.js/Python splice), Phase C (verify and clean up). |
| Mixing standard mode extraction with two-file mode | If user specifies exactly two `.md` files (one benchmark + one analysis), use Two-File Mode exclusively. Do not apply 3-field extraction rules designed for standard mode. |
| Truncating or summarizing code blocks from analysis file | Preserve ALL code blocks exactly as they appear in the analysis file. Never abbreviate, elide, or summarize code. |
| Omitting code blocks from the analysis card | The analysis card MUST include all code blocks from the source file. Code comparison (old vs new) is the core value of the analysis. |
| Using multi-series grouped/stacked bars to compare groups on different metrics | One chart = one metric = one comparison question. Split into per-metric charts (e.g., separate P50 chart, P99 chart) each with one bar per group. Multi-series bar charts obscure the comparison between groups. |
| Table font size below 17px | All table cell text must be ≥ 17px. "Tight" layouts with smaller fonts are unreadable. |
| Placing detailed code analysis (code blocks, diffs, breakdowns) in the main body | Code analysis is reference material — place it in the 附录 (appendix) at the end of the report. Only exception: user explicitly requests inline code analysis. Main body has: data + charts + conclusions. |

## Example Invocation

**Standard mode (directory scan):**

User: "帮我处理 /home/user/experiments/ 目录，生成实验汇报"

You:
1. List files in `/home/user/experiments/`
2. For each `.md`: run extraction (rules → LLM fallback)
3. For each data file: parse + select chart type
4. Fetch/embed ECharts source
5. Render HTML with all cards
6. Write to `/home/user/experiments/report.html`
7. Confirm: "已生成 report.html，包含 N 个实验卡片，M 个图表"

**Two-File mode (benchmark + analysis):**

User: "根据 BENCHMARK_REPORT.md 和 lagacy_fast_optimization_analysis.md 生成报告"

You:
1. Read both files, identify which is benchmark and which is analysis
2. From benchmark: extract experiment name, config, comparison groups (A/B/C), average rows, comparison pair tables, conclusions
3. From analysis: extract title, overview, section-by-section body, all code blocks, optimization breakdown table, core principles
4. Prepare chart data from benchmark comparison group average rows (QPS, latency, time breakdown)
5. Fetch/embed ECharts source
6. Render HTML with **main body + appendix**:
   - Card 1: Benchmark Results (data, per-metric charts, conclusions)
   - Card 2: Combined Summary (synthesized findings + brief optimization approach)
   - 附录: 代码详细分析 (full analysis with all code blocks preserved exactly)
7. Write `report.html` to the same directory as the input files
8. Confirm: "已生成 report.html，包含 Benchmark 数据卡片、综合总结和文末代码分析附录"
