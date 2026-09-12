---
name: r-research-code-style
description: Use when writing, reviewing, refactoring, or explaining R scripts for agricultural, climate, spatial, econometric, DSSAT, SEM, machine learning, or ggplot-based research workflows. Defines the user's preferred R writing style — detailed Chinese comments, traceable object naming, visible script sections, readable layout, and a fixed order for choosing between package functions and hand-written code — while deliberately leaving the specific technical choice open.

agent_created: true
---

# R Research Code Style

## Overview

This skill defines how the user's research R code should **read**, not which tools it should use.

The goal is not package-style R code. Preserve the practical research-script workflow, but make every script easier to read, rerun, audit, and revise for a paper. The measure of success is simple: the user should be able to reopen a script months later and reconstruct the research logic from the comments alone.

## Guiding Principle: Principles Over Prescriptions

**This skill specifies expressive style and structure. It does not specify technical implementation.**

- When several approaches are technically valid, choose the one that best fits the surrounding project, and state the trade-off in a Chinese comment. Do not defer to a default package, framework, or idiom just because it is mentioned here.
- Treat any concrete function name, package, or storage format appearing in this file as **illustration only** — it demonstrates comment density and layout, not a recommendation.
- Prefer a better solution to a prescribed one. If an approach not described here serves the research goal better, use it and explain why in the comment.
- Prescriptions appear in this skill only where the alternative is a **correctness or reproducibility risk**, not a matter of taste.
- Consistency with the existing script beats abstract best practice. When refactoring, match the file's established conventions unless the user asks to change them.

## Dependency Judgment Order

When a task needs a capability, consider candidates in this order. Reach the next tier only when the current one genuinely fails.

1. **Base R, or a package already loaded in this script.** No new dependency, nothing to verify.
2. **A well-established package in the same domain.** "Well-established" means it has a long CRAN history, is widely cited in published work and existing data pipelines, and is still maintained. Prefer it over hand-written code even when the hand-written version is short.
3. **An unfamiliar or thinly-used package.** Inspect the function's implementation before trusting it: read the source, and check the algorithm it actually uses, its boundary and missing-value behaviour, its units, and whether it vectorises. Write the conclusion of that review in a Chinese comment, including what you verified and any residual doubt.
4. **Write it yourself** — only after the first three tiers fail. Say in the comment which candidates were rejected and why: not vectorised, wrong units or reference system, too heavy a dependency, insufficient precision, or unmaintained. State the key assumptions of the hand-written version.

Two failure modes to avoid, in both directions: do not hand-write something a mature package already solves correctly, and do not introduce an unfamiliar dependency to replace three lines of transparent, verified code.

Performance is a legitimate reason to leave tier 1–3. Established functions are often scalar point-wise interfaces; calling one per row across a million-row panel is orders of magnitude slower than a vectorised alternative. When that is the case, vectorisation — including a hand-written vectorised implementation — is the correct choice, and the comment should record the trade-off rather than hide it.

## Replacing An Implementation Is A Semantic Change

Swapping one implementation for another is never a pure upgrade. A more accurate or more standard function can silently change the meaning of every downstream result. After any replacement, verify each of these explicitly:

- **Units.** Length in metres vs kilometres, area in hectares vs square metres, rates in percent vs fraction. This is the most common silent break.
- **Order and convention of arguments**, e.g. longitude before latitude, or `(y, x)` instead of `(x, y)`.
- **Reference model**, e.g. spherical approximation vs ellipsoid, which shifts long-distance values by a small but non-zero amount.
- **Missing-value behaviour** — whether `NA` propagates, is dropped, or is returned as `NaN`.
- **Return type and shape**, e.g. a scalar vs a length-1 vector, a matrix vs a vector, a `units`-classed object vs a bare numeric.
- **Vectorisation**, i.e. whether a call works element-wise on the whole column or only on a single point.

Either make the difference disappear (convert back to the original unit) or state it where the reader will meet it. **A function that returns a physical quantity must carry its unit in the name, the argument names, or an adjacent comment.** A bare name like `distance` is not acceptable.

```r
# 反例：返回值单位未交代，调用方无法判断是米还是公里，也无法察觉替换实现后的量纲变化
ec_distance <- function(lon, lat, report_lon, report_lat) {
  geosphere::distGeo(c(lon, lat), c(report_lon, report_lat))
}

# 正例：单位进入函数名，并交代为何采用成熟包而非手写，以及代价是什么
# 返回：站点到报告点的椭球距离，单位 km
ec_distance_km <- function(lon, lat, report_lon, report_lat) {
  # 采用成熟测地包而非手写球面近似：椭球模型在大尺度上更准确，且省去自行处理数值边界。
  # 代价是该接口为单点标量，不向量化，故仅用于点位级调用，勿直接用于百万行面板。
  # 坐标顺序为 (经度, 纬度)；原包返回米，此处统一除以 1000 换算为公里。
  meters <- geosphere::distGeo(c(lon, lat), c(report_lon, report_lat))
  meters / 1000
}
```

## Voice And Tone

Comments should sound like a researcher's reasoning, not a code annotation.

- Lead with purpose: state what question this step answers, or what problem it prevents.
- Prefer causal phrasing over descriptive phrasing — "为避免回归自动删样本导致样本量不可控" rather than "删除缺失值".
- Record the things that are invisible in code: units, sample size, thresholds and where they came from, variable interpretation, projection, how a coefficient or plotted layer should be read.
- Name remaining assumptions and caveats in place, where a future reader will meet them.
- Do not write comments that instruct the reader, narrate obvious syntax, or repeat the function name.
- When a choice is tentative, mark it explicitly so it is easy to find later.

```r
# Bad: 删除 NA
df_model <- df_raw |> filter(!is.na(Yield))

# Good: 删除产量缺失记录，避免模型自动删样本导致样本量在不同设定下不可比
df_model <- df_raw |>
  filter(!is.na(Yield))

# TODO: 该异常值阈值目前凭经验设定，正式分析前需用分位数或文献依据重新确认
yield_clean <- yield_raw |>
  filter(Yield < 3e4)
```

## Core Rules

- Write detailed Chinese comments that explain research intent, not just syntax.
- Use clear object names that describe data source, crop, model, period, or output purpose.
- Put packages, paths, years, thresholds, variable lists, and output names near the top.
- Prefer `TRUE` and `FALSE` over `T` and `F`.
- Never overwrite raw input objects in place when the next steps depend on traceability; write derived results to new objects and new output files.
- Avoid absolute paths inside analysis logic. If a path must be absolute, define it once in the parameter section with a Chinese comment.
- After grouped mutation or summarisation, call `ungroup()` unless grouped behavior is intentionally needed later.
- Separate data preparation, modeling, diagnostics, plotting, and output into visible sections.
- Match the surrounding file's conventions for assignment operator, pipe operator, quoting, and namespace style. In a new script, pick one convention and hold it to the end.
- Prefer an established package function over hand-written code for any non-trivial computation, including short formulaic ones. Hand-written versions are the last resort, not the default for brevity.
- Make units part of the name or the comment for every object holding a physical quantity. Unit-carrying helper functions must state their output unit in the name or an adjacent comment.
- When the project fixes a convention for units, coordinate order, or reference system, follow it even if a candidate function defaults to something else; convert at the boundary rather than letting two conventions coexist.

## Script Structure

Use this default structure for new R scripts:

```r
# 0. 脚本说明 ----------------------------------------------------------------
# 研究目的：
# 输入数据：
# 输出结果：
# 主要方法：
# 注意事项：

# 1. 加载 R 包 ----------------------------------------------------------------

# 2. 参数设置 -----------------------------------------------------------------

# 3. 自定义函数 ---------------------------------------------------------------

# 4. 数据读取 -----------------------------------------------------------------

# 5. 数据清洗与变量构造 -------------------------------------------------------

# 6. 模型构建或空间分析 -------------------------------------------------------

# 7. 结果诊断与稳健性检查 -----------------------------------------------------

# 8. 绘图与结果输出 -----------------------------------------------------------
```

For short snippets, still include enough structure to show where the code belongs.

## Naming Style

Use names that help the user remember the object after several months.

| Object type  | Prefer                                  | Avoid                                    |
| ------------ | --------------------------------------- | ---------------------------------------- |
| Raw data     | `yield_maize_raw`, `weather_county_raw` | `data`, `df`                             |
| Clean data   | `yield_maize_clean`                     | overwriting `yield_maize_raw`            |
| Model data   | `yield_maize_model_df`                  | `test`, `temp`                           |
| Model object | `model_fe_wind_temp`, `rf_yield_model`  | `model`, `result` when many models exist |
| Plot object  | `plot_wind_margin`, `fig3_waterfall`    | `p1`, `p2` unless assembling a figure    |
| Output table | `table_fe_robust_se`                    | `out`, `res`                             |

Short names such as `p1`, `p2`, `p3` are acceptable only inside a figure assembly block, and each must have a nearby comment explaining its panel meaning.

## Readability Rhythm

Layout is part of the style. Keep the page calm and scannable.

- One step per line inside a pipeline; break the chain rather than letting it run off the edge.
- Wrap long function calls so arguments align vertically and each argument can carry its own short comment.
- Use a blank line between semantic blocks, not between every line.
- Keep section banners visually uniform so the outline is visible when scrolling fast.
- Prefer a few clearly named intermediate objects over one dense expression — the reader should never have to parse a nested call to understand intent.
- Extract a repeated block into a helper function only when the repetition is meaningful; a small amount of duplication reads better than premature abstraction.
- Do not leave commented-out code, debugging leftovers, or abandoned variants in the body.

```r
# 构造县级玉米分析样本：气象距平与产量匹配后，仅保留同时具备两类信息的县-年记录
yield_maize_model_df <- yield_maize_clean |>
  left_join(
    weather_county_growth,
    by           = c("county_id" = "adcode", "Year" = "year"),  # 县码 + 年份双键，避免跨年错配
    relationship = "many-to-one"                                # 显式声明匹配关系，使异常一对多立即报错
  ) |>
  filter(!is.na(wind_growth_anomaly))

# 检查未匹配记录数；非 0 说明编码口径或年份对齐需要回到源表核对
sum(is.na(yield_maize_model_df$wind_growth_anomaly))
```

## Narrative Requirements By Stage

Each stage has one dominant obligation: make its logic legible. How that is achieved is open.

1. **Reading and cleaning** — state what each removed or transformed record means for sample definition, and check the result of the operation.
2. **Joining** — make keys explicit, and verify what failed to match.
3. **Modeling** — write the research question and the identifying assumption before the specification; explain each non-obvious term in Chinese; keep diagnostics next to the model they diagnose.
4. **Spatial and raster work** — state the coordinate reference system and the unit before transforming; explain why a rotation, reprojection, mask, or weighting step is needed.
5. **Plotting** — name each figure by its role, comment non-obvious geoms, scales, thresholds, and annotations, define shared theme values once, and keep export calls at the end of the section.
6. **Output** — keep exported objects distinguishable from inputs, and prefer writing a new output file over replacing an existing one.

## Paths And Outputs

Define paths once and keep outputs organized:

```r
# 项目根目录：所有输入与输出路径都从这里拼接，便于迁移到其他电脑
project_dir <- "<在这里填写项目根目录>"

data_dir   <- file.path(project_dir, "data")
output_dir <- file.path(project_dir, "output")
figure_dir <- file.path(output_dir, "figures")
table_dir  <- file.path(output_dir, "tables")

dir.create(figure_dir, recursive = TRUE, showWarnings = FALSE)
dir.create(table_dir,  recursive = TRUE, showWarnings = FALSE)
```

Do not scatter drive-letter paths through the script body. If the user provides existing absolute paths, keep them in the parameter section and explain what each folder contains.

## Review Checklist

When reviewing or refactoring R code, report issues in this order:

1. Reproducibility risks: hard-coded paths, hidden workspace objects, missing packages, encoding problems.
2. Data risks: silent row deletion, unclear join keys, unit conversion without comments, overwritten raw data.
3. Statistical risks: unexplained model terms, missing diagnostics, unclear sample definition.
4. Implementation risks: a hand-written routine where an established package exists; an unfamiliar package used without inspecting its source; a replacement that silently changed units, argument order, reference model, missing-value behaviour, or vectorisation.
5. Readability risks: vague names, repeated blocks, very long scripts, scattered output calls.
6. Style issues: `T/F`, inconsistent convention, duplicated `library()`, mixed pipes, commented-out clutter.

Give an improved code version when possible, not just criticism. Keep the tone constructive: preserve the user's research logic while making the script more reliable. Do not rewrite working logic into a different technical approach unless the user asked.

## Before Returning Code

- Check that comments are in readable Chinese and not mojibake.
- Check that every major analysis step has a purpose comment.
- Check that raw, clean, model, plot, and output objects are distinguishable.
- Check that no unexplained absolute paths remain in the middle of the script.
- Check that non-trivial computations go through an established package, or that a hand-written implementation is justified in a comment.
- Check that every physical quantity crossing a function boundary has a stated unit, and that any replacement of one implementation by another has been reconciled for unit, argument order, reference model, missing values, and vectorisation.
- Mention any assumptions the code makes about file names, column names, units, or packages.

## What This Skill Deliberately Does Not Specify

These are left to judgment at execution time. Choosing differently is correct when the project context calls for it:

- Which specific package or estimator to use. What is specified is the order in which to consider candidates and the obligation to review an unfamiliar one — not the candidate itself.
- How to parallelise, cache, log, or monitor long-running work.
- Which file format to use for intermediate or exported objects.
- Whether to abstract repeated code into functions or packages.
- How to manage environments, seeds, or reproducibility infrastructure beyond stating that stochastic steps need a seed.
