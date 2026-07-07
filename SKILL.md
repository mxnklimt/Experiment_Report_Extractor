---
name: experiment-report-extractor
description: Use when user provides a directory of experiment data/logs (Markdown + data files, or pure data files) and wants a self-contained HTML report. Before ANY generation, this skill iteratively confirms with the user: file scope → key metrics → chart preferences → per-section report content. Nothing is generated until approved. Trigger when user mentions "实验报告", "生成HTML", "提取实验信息", "汇总数据", "report", "benchmark report", or provides a directory of experiment logs.
---

# Experiment Report Extractor

Turn experiment data and logs into a self-contained HTML report through iterative confirmation dialogue.

Start by scanning the input directory/files, then ask questions one at a time to understand what matters most. Once the user's priorities are clear, present a report blueprint for approval before generating anything. Generate the report section by section, getting approval after each.

## Anti-Pattern: "This Is Simple Enough To Auto-Generate"

Every report goes through this process. A single CSV with 3 columns, a directory of markdown logs — all of them. "Simple" reports are where unexamined assumptions about what the user cares about cause the most wasted work. The confirmation can be short (one exchange for truly simple inputs), but you MUST confirm before generating.

## Core Principle: AI-First Analysis → User Alignment

Before asking the user "what's the conclusion?" or "what's important here?", always do your own analysis first. Read the data, identify patterns, compare numbers across groups, and form a clear hypothesis about the core finding. Then present your analysis to the user and ask them to confirm, correct, or refine it.

This applies at every confirmation point:

| Step | AI-first behavior |
|------|-------------------|
| **3c** (pure data) | Scan the data yourself → propose what you think the core finding is → ask user to align/edit |
| **3d** (extraction verify) | Present your own data-driven analysis alongside the .md extracted text → user aligns both |
| **5** (blueprint) | Blueprint MUST include your proposed global conclusion, NOT "(to be generated in Step 6)" |
| **6** (generation) | Global conclusion block: present AI's analyzed conclusion first → user confirms/edits before proceeding |

The user is hiring you to do the intellectual work of reading and analyzing data — finding the story in the numbers. Don't hand them an empty form and ask them to fill it in. They already have the data; your job is to discover and articulate what it means, then let them refine.

**How to analyze data for conclusions:**

1. Compare the key metrics across groups — which group performs best? By how much?
2. Look for patterns: which metric shows the biggest difference? Which shows none?
3. Trace causality: if one metric improved dramatically, what likely caused it?
4. Quantify: always use specific numbers and percentages, not vague "improved" statements
5. Be honest about uncertainty: if the data doesn't clearly support a conclusion, say so

**Example of good AI-first analysis:**

```
I've looked through results_a.csv. Here's my analysis:

  Group C vs Group A comparison:
    QPS:        29.21 vs 26.14 → C is +11.7% higher
    Avg latency: 136.3 vs 148.7 → C is 8.3% faster
    P99 latency: 142.1 vs 157.3 → C's tail latency is 9.7% better
    Concat time: 3.4 vs 15.1  → C's concat is 4.4× faster ← biggest contributor
    Infer time:  115.9 vs 118.6 → negligible difference (~2.3%)

My proposed core conclusion:
"Group C leads in both throughput (+11.7%) and latency (-8.3%), with concat
optimization (15.1ms → 3.4ms) as the primary driver. Inference time is nearly
identical across groups, confirming the gain is in the data pipeline, not the model."

Does this match your understanding? Feel free to correct, refine, or add context.
```

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Scan & scope** — scan input directory/files, list all files, confirm report scope
2. **Discover data structure** — read each data file/Markdown, list all columns/fields/sections
3. **Confirm key metrics** — per file/experiment: which columns in body vs appendix, experiment names, group labels, background info
4. **Confirm chart preferences** — chart types, comparison dimensions, Y-axis baseline, layout density
5. **Propose report blueprint** — aggregate all confirmations into a "report blueprint" — the final gate before generation
6. **Generate & present per-section** — generate body content section by section, approve each before proceeding (global summary → per-card conclusion+table → per-card charts)
7. **Generate appendix** — generate appendix (full data, code analysis), approve
8. **Full report self-review** — holistic self-check (data accuracy, chart completeness, conclusion consistency, style, offline readiness)
9. **User final review** — user final review → deliver `report.html` upon approval

**The terminal state is delivering `report.html`.**

## Process Flow

```
Scan & Scope
  ↓
Discover Data Structure
  ↓
Confirm Key Metrics ←── revise ──┐
  ↓                              │
Confirm Chart Preferences ←─ revise ─┐
  ↓                              │
Report Blueprint ←──── revise ───┘
  ↓ (HARD-GATE unlocks)
Generate Section 1 → [Approve? ─No─→ revise]
  ↓Yes
Generate Section N → [Approve? ─No─→ revise]
  ↓Yes
Generate Appendix → [Approve? ─No─→ revise]
  ↓Yes
Self-Review → User Final Review → Deliver
```

## HARD-GATE

```
BEFORE Step 5 (Report Blueprint) is explicitly approved by the user, it is FORBIDDEN to:
  - Write any HTML code
  - Generate any CSS styles
  - Configure any ECharts charts
  - Assemble any data tables

All that is ALLOWED before the gate:
  - Read files, parse data, extract fields
  - Conduct confirmation dialogue with the user
  - Record user choices and preferences
```

After the blueprint is approved, generation proceeds step by step — each section must be approved before the next begins. Skipping approval on any section is forbidden.

## The Process

### Step 1 — Scan & Scope

**Two input modes:**

| Mode | Input | Behavior |
|------|-------|----------|
| Directory mode | Directory path | Scan all `.md` + `.csv`/`.tsv`/`.json`/`.jsonl`/`.xlsx` |
| File mode | 1~N file paths | Use the user-specified file list directly |

**Scan rules:**
- Collect `.md` files → experiment documents (optional, may be absent)
- Collect `.csv` `.tsv` `.json` `.jsonl` `.xlsx` → data sources
- Exclude binaries, images, `.html`, and other non-data files

**Present to user for confirmation:**

```
Found N Markdown documents, M data files:

Markdown:
  1. benchmark_report.md (12KB)
  2. analysis.md (8KB)

Data files:
  3. results_a.csv (45KB)
  4. config.json (1KB)

Include all in the report? Any to exclude?
```

Wait for user confirmation before proceeding to Step 2. Only proceed once scope is confirmed.

### Step 2 — Discover Data Structure

**For each data file:**
- Parse headers → list all column names
- Show first 3 rows as sample data
- Auto-detect column types (numeric/categorical/temporal/text)
- Report row count

**For each Markdown file:**
- List section heading tree (`#`, `##`, ...)
- Annotate table locations and code block counts

**Present format:**

```
📄 results_a.csv — 8 columns, 200 rows

  Column                         Type         Sample
  ───────────────────────────────────────────────────
  timestamp                      temporal     2026-07-01 14:00:01
  group                          categorical  A / B / C
  qps                            numeric      27.69
  avg_latency_ms                 numeric      143.9
  p50_latency_ms                 numeric      138.2
  p99_latency_ms                 numeric      144.2
  concat_time_ms                 numeric      15.1
  infer_time_ms                  numeric      115.9

📄 benchmark_report.md — 3 sections
  # GPU Benchmark Report
    ## Experiment Config    (1 table)
    ## Detailed Results     (3 tables)
    ## Conclusion           (plain text)
```

Present one file at a time. Wait for the user to acknowledge each before moving on.

### Step 3 — Confirm Key Metrics (core confirmation step)

**One file/experiment at a time.** Multiple files → go through each sequentially.

This is the most critical step. For each data file/experiment, confirm three things with the user:

#### 3a. Body vs Appendix column split

Present all columns and let the user check which ones go into the report body (featured prominently) vs appendix (full data preserved at end):

```
📄 results_a.csv — Which columns should appear in the body (★)?

  [★] qps               → body
  [★] avg_latency_ms    → body
  [★] p99_latency_ms    → body
  [ ] timestamp          → appendix
  [ ] group              → appendix (use as group label)
  [ ] p50_latency_ms    → appendix
  [★] concat_time_ms    → body
  [★] infer_time_ms     → body
```

Default: all numeric columns are candidate for body. User checks the ones they care about. Unchecked columns go to appendix.

#### 3b. Experiment naming + group labels

If the data has a categorical column (e.g., `group` = A/B/C):

```
Detected 3 values in group column: A, B, C

Please name each group (used for chart X-axis labels):
  A → [A: baseline              ]
  B → [B: vectorized inference  ]
  C → [C: vectorized + concat   ]

Experiment name: [GPU Benchmark] (editable, used as report title)
```

#### 3c. Pure data mode — additional confirmation

When NO `.md` files are present:

**CRITICAL — Do NOT ask the user to fill in a blank form.** First, analyze the data yourself and propose a conclusion, THEN ask the user to align.

```
No Markdown documents found, so I've analyzed the data directly.

Based on results_a.csv, here's my analysis:

  [AI analyzes each body metric across groups, comparing best vs worst,
   quantifying differences with specific numbers and percentages]

My proposed core conclusion:
"[1-2 sentences stating the most important finding, with specific numbers]"

Does this match your understanding? Please correct or refine:
  Experiment background/purpose: [                              ]
  Key conclusion (align/edit):   [pre-filled with AI's proposal]
```

The AI's proposed conclusion gives the user something to react to — they can say "yes, that's right" or "no, actually the key point is X." Either way, the user doesn't start from a blank page.

#### 3d. Extraction verification (when `.md` files exist)

If the skill extracted name/results/conclusion from Markdown, present them alongside your own data-driven analysis:

```
📄 benchmark_report.md extraction + my data analysis:

  .md extracted name:  [GPU Benchmark Report]              ✓ / ✗ edit
  .md extracted result: [Group C highest QPS 29.21]         ✓ / ✗ edit
  .md extracted conclusion: [Vectorization + concat → +5.5%] ✓ / ✗ edit

  Based on my own analysis of results_a.csv:
  "Group C leads at 29.21 QPS (11.7% above A), driven by concat dropping
   from 15.1ms to 3.4ms. Inference time is flat — the gain is pure pipeline."

  → .md extraction and data analysis: [consistent / have differences — see below]
```

When the .md extraction and your data analysis disagree: flag the discrepancy explicitly and ask the user to resolve. The data doesn't lie — if the .md says "no significant improvement" but the data shows +11.7%, surface that tension.

When they agree: the user gets double confirmation that both sources point to the same conclusion.

**Important:** Ask one file's worth of questions at a time. Do not batch multiple files' column selections together.

### Step 4 — Confirm Chart Preferences

Based on the body columns the user selected in Step 3, propose charts and confirm preferences:

```
Based on your selected body metrics, suggested charts:

Body charts (one comparison bar chart per metric):
  QPS comparison         → bar chart (A vs B vs C)
  Avg latency comparison → bar chart (A vs B vs C)
  P99 latency comparison → bar chart (A vs B vs C)
  Concat time comparison → bar chart (A vs B vs C)
  Infer time comparison  → bar chart (A vs B vs C)

Options:
  [ ] Need pairwise comparison charts (A vs B / B vs C)?
  [ ] Any time-series data that needs line charts?
  [ ] Chart layout: 1 large per row / 2-column grid / 3-column grid
  [ ] Y-axis baseline: always from 0 / auto-range / non-zero when diff < 10%

Any metrics that should use scatter or other chart types?
```

**Rules:**
- Default: each metric = one bar chart comparing all groups
- Default: 2-column grid layout
- Default Y-axis: non-zero only when max-min diff < 10%
- User can override any default

### Step 5 — Report Blueprint (final gate before HARD-GATE unlocks)

Aggregate ALL confirmations from Steps 1-4 into a single "report blueprint". This is the LAST confirmation before any HTML generation begins.

```
📋 Report Blueprint — report.html

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 Overview
  · Experiment cards: 2
  · Body charts: 10
  · Data sources: results_a.csv, results_b.csv, benchmark.md
  · Mode: directory mode (with .md)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 Global Summary Block
  AI's proposed core conclusion:
  "[1-2 sentence synthesis of the most important finding across all experiments,
   based on data analyzed in Steps 2-3]"

  (User can edit this now, or refine during Step 6 generation)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🃏 Card #1: GPU Benchmark
  Source: benchmark_report.md + results_a.csv
  Section conclusion (to be generated in Step 6)

  Body display:
    QPS comparison    | bar chart | Y-axis auto-range
    Avg latency       | bar chart |
    P99 latency       | bar chart | diff < 10%, non-zero Y
    Concat time       | bar chart |
    Infer time        | bar chart |

  Appendix contains: timestamp, p50, group columns + full data

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📎 Appendix (end of report)
  Appendix A: Full data tables (all filtered columns)
  Appendix B: Code analysis (if analysis.md present)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎨 Style
  Font: body 17px, chart label 16px
  Chart height: ≥ 400px
  Chart layout: 2-column grid

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Approve this blueprint to begin section-by-section generation.
Reject → return to Step 3 or 4 to adjust.
```

**If user approves → HARD-GATE unlocks → proceed to Step 6.**
**If user rejects → return to the step they want to adjust.**

Do NOT under any circumstances begin HTML generation before blueprint approval.

### Step 6 — Generate & Present Per-Section

HARD-GATE is now unlocked. **Generate ONE section at a time. Approve each before proceeding to the next.**

**Fixed generation order:**

```
Global conclusion block → AI presents analyzed conclusion → user aligns/edits → approve
  ↓
Card #1 section conclusion + data table → approve
Card #1 charts → approve
  ↓
Card #2 section conclusion + data table → approve
Card #2 charts → approve
  ↓
... (repeat until all cards complete)
```

**Global conclusion block — AI-first pattern:**

The global conclusion is the most important text in the report. Don't make the user write it from scratch. Before generating the HTML, present your proposed conclusion:

```
Based on all data analyzed, here's my proposed global conclusion for the report:

"[1-2 clear, specific sentences with numbers — the single most important finding
  the reader should take away]"

Supporting evidence:
  · [key metric comparison with specific numbers]
  · [key metric comparison with specific numbers]
  · [causal explanation if identifiable]

→ Does this capture it? Edit as needed, then I'll generate the HTML.
```

Once the user confirms the conclusion text, generate the `class="global-summary"` HTML block with that text. Do NOT generate the block with placeholder text — the conclusion must be real and confirmed before it goes into HTML.

**Approval prompt after each section:**

```
Generated [Card #1 — GPU Benchmark] section conclusion and data table.

✓ Approved, continue
✗ Revise: [user specifies what to change]
```

**Revision rules:**
- Current section not approved → revise → re-present → re-approve, loop until approved
- If a later change affects an already-approved earlier section → proactively inform the user, wait for confirmation before back-editing
- User can say "go back to Card #1" at any time to revise approved content
- User can say "skip charts for this card" to jump to the next card

**Card content differs by mode:**
- With `.md`: section conclusion + extracted result summary + charts + original conclusion citation
- Pure data: section conclusion (from user-provided context) + charts + brief data summary

**Section conclusions per card — also AI-first:** The same principle applies at the card level. For each card, propose a 1-sentence section conclusion based on that card's data before generating HTML. The user confirms, then you generate. Don't make the user write section conclusions either.

**When generating charts and HTML, follow ALL style specifications in the Appendix below.** This includes ECharts inline process, font sizes, chart dimensions, anti-overlap rules, per-metric comparison principle, conclusion-first principle, and all other existing specifications.

### Step 7 — Generate Appendix

After ALL body cards are approved, generate the appendix:

```
📎 Appendix content:

  · Full data tables — all filtered columns, complete display
  · Code analysis (if analysis.md present) — all code blocks preserved verbatim
  · Appendix conclusion block — one sentence summarizing appendix core content

Appendix opening conclusion: [to be generated, confirmed during approval]
Appendix layout: single-column wide tables
```

**Appendix approval** follows the same flow as body sections — revise until approved.

**Two-file mode appendix rule:** When the input includes a benchmark + analysis pair, the code analysis goes into the appendix (not body), with all code blocks preserved exactly. Only exception: user explicitly requests code inline in body.

### Step 8 — Full Report Self-Review

After all body sections and appendix are generated and approved, perform a holistic self-check. **Report any issues to the user before fixing them. Do not silently correct.**

**Self-review checklist:**

| Category | Check |
|----------|-------|
| Data accuracy | Every number matches source file; body and appendix data are consistent |
| Chart completeness | Every chart renders correctly; Y-axis/labels/legends correct; no overlaps |
| Conclusion consistency | Global conclusion ↔ section conclusions ↔ data are consistent; no contradictions |
| Style compliance | Font ≥ 17px, chart ≥ 400px, no text overlap, ECharts inline |
| Offline readiness | Single `.html` file, zero CDN dependencies, no broken paths |
| Appendix completeness | All filtered columns present in appendix; code blocks complete without omission |

Fix any issues found, then re-run the checklist items relevant to the fix. Only proceed to Step 9 when all items pass.

### Step 9 — User Final Review & Deliver

```
✅ Report self-review passed.

Complete report generated: report.html

Contains:
  · N experiment cards
  · M ECharts charts
  · 1 complete appendix

Please review the full report. Tell me what to revise (location + content).
When all confirmed, the report is delivered.
```

**After delivery:**
- Do NOT commit to git, open browser, or take any extra action — unless the user explicitly requests it.

## Transition Rules

The terminal state of this skill is **delivering `report.html`**. Unlike brainstorming, this skill does NOT automatically transition to `writing-plans` or any other skill.

If the user says "stop here, save this design as a spec" during the approval process — respond flexibly, do not continue generating HTML.

If the user wants to re-generate a report with different parameters, start from Step 1 again.

## HARD-GATE Detail

| Phase | Before gate: ALLOWED | Before gate: FORBIDDEN |
|-------|---------------------|----------------------|
| Steps 1-4 | Read files, parse data, extract fields, confirmation dialogue | Generate ANY HTML/CSS/JS |
| Step 5 | Aggregate blueprint, user approval | Same as above |
| Steps 6-7 (gate passed) | Generate HTML fragments, per-section approval | Skip approval and generate next section |
| Steps 8-9 | Self-review, fix, deliver | Claim completion without user final confirmation |

---

# Appendix: Visual & Data Specifications (Preserved)

The following specifications from the original skill are fully preserved. Follow these when generating HTML in Steps 6-7.

## HTML Structure

### Global & Section Conclusion Blocks

All reports MUST follow "conclusion first" principle:

- **Global conclusion block** (`class="global-summary"`): placed right after the page title, before any data or charts. One sentence stating the most important finding of the entire report.
- **Section conclusion block** (`class="section-summary"`): placed at the start of each card body, before data/charts. One sentence stating that section's conclusion.

```css
.global-summary {
  background: #e8f4fd;
  border-left: 5px solid #4a90d9;
  border-radius: 6px;
  padding: 16px 24px;
  margin: 20px 0 28px 0;
  font-size: 17px;
  line-height: 1.8;
}
.section-summary {
  background: #f0f7ee;
  border-left: 4px solid #52a652;
  border-radius: 4px;
  padding: 12px 18px;
  margin: 0 0 20px 0;
  font-size: 16px;
  line-height: 1.7;
}
```

### Conclusion Writing Rules

| Requirement | ❌ Wrong | ✓ Right |
|------------|---------|---------|
| Clear | "Some interesting observations from the experiment" | "Group C QPS is 5.5% higher than A" |
| Direct | "There are some differences between X and Y" | "X is 23% faster than Y, mainly from concat dropping from 15ms to 3.4ms" |
| Understandable | "Vectorized batch inference reduced latency tail" | "Batching inference requests reduced P99 latency from 144ms to 136ms" |

## Readability Specifications (MANDATORY)

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

## ECharts Inline Process (CRITICAL)

Use a placeholder-based two-phase assembly to avoid PowerShell `$` variable corruption:

**Phase A:** Write the HTML skeleton with `__ECHARTS_LIBRARY_GOES_HERE__` placeholder.
**Phase B:** Use Node.js to splice the real ECharts source:

```js
const fs = require('fs');
const html = fs.readFileSync('report.html', 'utf8');
const echarts = fs.readFileSync('echarts.min.js', 'utf8');
fs.writeFileSync('report.html', html.replace('__ECHARTS_LIBRARY_GOES_HERE__', echarts), 'utf8');
```

**Phase C:** Verify exactly 2 `<script>` tags and placeholder is gone. Delete intermediate files.

**NEVER use PowerShell `-replace` or here-strings to splice ECharts** — dollar-variable expansion CORRUPTS JavaScript.

## Anti-Overlap Rules (MANDATORY)

### X-Axis Labels

- X-axis labels MUST be horizontal (`axisLabel.rotate: 0`). Rotated/slanted text is FORBIDDEN.
- When > 6 X-axis categories, split into multiple smaller charts (≤ 6 categories each).
- Abbreviate labels to ≤ 6 characters (e.g. "A: baseline" → "A:base"). Full name in tooltip.
- For labels that cannot be abbreviated, use `axisLabel.overflow: 'truncate'` and `axisLabel.width: 80`.
- Set `axisLabel.interval: 0` — never let ECharts auto-skip labels.
- Charts with 5+ categories MUST be full-width (no grid column).

### Data Labels on Bars

- Dense charts (> 6 bars): hide labels (`label.show: false`), use tooltip.
- ≤ 6 bars: show labels above bars (`label.position: 'top'`), ensure chart height is tall enough.

### Grid Margins

```js
grid: {
  top: 55,    // space for legend/title (75 if legend has many items)
  right: 45,  // space for data labels on rightmost bar
  bottom: 65, // space for horizontal x-axis labels
  left: 80    // space for y-axis labels
}
```

### Y-Axis Baseline for Bar Charts

- When `(max - min) / max < 0.10`: Y-axis MUST NOT start from 0.
- Set `yAxis.min` to `min × 0.85` (rounded to clean tick value).
- Always add annotation: `<div style="font-size:12px;color:#95a5a6;text-align:center;margin-top:-8px;">Note: Y-axis starts at {min} to highlight small differences</div>`.
- When diff ≥ 10%: keep `yAxis.min: 0`.

## Per-Metric Comparison (MANDATORY)

**One chart = one metric = one comparison question.** NEVER bundle multiple metrics into one chart.

| ❌ Wrong | ✓ Right |
|---------|---------|
| 1 chart with 5 series (Avg/P50/P90/P95/P99) × 3 groups | 5 separate charts, each comparing 3 groups on one metric |

Each chart's X-axis = comparison groups (A, B, C). Y-axis = the single metric.

## HTML Content Overlap Prevention (MANDATORY)

- `table-layout: fixed` on all `<table>`, percentage column widths, `word-break: break-word` on `<td>`
- `td { vertical-align: top; }` for text-heavy columns
- Card body: `padding: 20px 30px` minimum, `margin-bottom: 18px` between consecutive sections
- Each `.chart-container`: `margin: 16px 0` minimum, never put two charts in one container
- Side-by-side charts: each container `min-height: 380px`
- Body text: `line-height: 1.7` minimum — NEVER use `line-height: 1` or `1.2`

## Output

Save as `report.html` in the same directory as the input files (or user-specified location).

## Two-File Mode (benchmark + analysis)

When user specifies exactly two `.md` files (one benchmark, one analysis):

### Main Body
- Card 1: Benchmark Results (config table + per-metric ECharts bar charts + conclusion)
- Card 2: Combined Summary (synthesized conclusion + brief optimization approach, NO detailed code)

### Appendix
- Full code analysis: all code blocks preserved verbatim, optimization breakdown table, core principles

### Code Block Rendering

```css
.code-block {
  background: #1e1e1e; color: #d4d4d4; padding: 16px 20px;
  border-radius: 8px; overflow-x: auto;
  font-family: 'Consolas', 'Courier New', monospace;
  font-size: 13px; line-height: 1.6; margin: 12px 0;
  white-space: pre; max-width: 100%; border: 1px solid #333;
}
.code-inline {
  background: #f0f0f0; color: #c7254e; padding: 2px 6px;
  border-radius: 3px; font-family: 'Consolas', 'Courier New', monospace;
  font-size: 0.9em;
}
```

## Common Mistakes

| Mistake | Correct |
|---------|---------|
| Using Chart.js or Plotly | **Always use ECharts** |
| CDN link for ECharts | **Inline full ECharts source** in `<script>` |
| Making charts too small (< 400px) | Chart height ≥ 400px, fonts ≥ 16px |
| X-axis labels overlapping | Abbreviate to ≤ 6 chars; split charts when > 6 categories. **NEVER rotate.** |
| Y-axis starts at 0 when diff < 10% | Set `yAxis.min` to `min×0.85`, add annotation |
| Rotated X-axis text | **Forbidden.** Abbreviate or split charts. |
| Table cells overflowing | `table-layout: fixed`, `word-break: break-word` |
| PowerShell `-replace` to splice ECharts | Use **Node.js or Python** placeholder-replace |
| Multi-series grouped bars comparing different metrics | One chart = one metric. Split into per-metric charts. |
| Code analysis in body instead of appendix | Code goes in appendix unless user explicitly says otherwise |
| No global conclusion block | MUST have `class="global-summary"` right after title |
| No section conclusion in cards | MUST have `class="section-summary"` at start of each card body |
| Vague conclusions ("some improvement") | Specific numbers required ("23% faster", "144ms→136ms") |
