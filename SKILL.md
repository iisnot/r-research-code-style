---
name: r-research-code-style
description: Use when writing, reviewing, refactoring, or explaining R scripts for agricultural, climate, spatial, econometric, DSSAT, SEM, machine learning, or ggplot-based research workflows. Defines the user's preferred R writing style — detailed Chinese comments, traceable object naming, visible script sections, readable layout, and a fixed order for choosing between package functions and hand-written code — while deliberately leaving the specific technical choice open. Also covers panel-data time-coverage metadata (why year_min/year_max is an interval and not coverage, carrying distinct-period sets as compressed contiguous segments, backfilling with a data.table update join rather than merge), multi-source normalization (taking admin level and measurement conventions from the source's own documentation instead of inferring them from the data, and checking existing crop keys before adding new normalization rules), and reading large external tables (haven col_select to pull only needed columns, per-variable NA screening, and deciding on caching by parse cost rather than file size).

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
- Treat the min/max of a time index as an **interval**, never as **coverage**. See the section below before naming any column `n_years`.

## 时间覆盖元数据：区间端点 ≠ 实际覆盖

面板数据做目录、索引、覆盖率统计时最容易出错的一点：`year_min` / `year_max` 只描述区间
端点，中间可能一片空白。本项目的实测（4 套次国家级作物统计，13,088 条
数据集×国家×作物×变量 记录，2026-10-02）：

- **29.4%** 的记录 `实际年份数 < 区间跨度`；
- FEWS NET 层面 **31%** 的国家×作物组存在断档；
- 极端案例：FAO HiH 赞比亚木薯标称 1976–2018（43 年），实际只有 13 年；
  AgroMaps 美国玉米标称 1970–2002（33 年），实际只有 4 年（1970、1987、2001–2002）；
  卢旺达玉米标称 1984–2017，实际只有 7 个年份。

**规则**：

1. 任何名为 `n_years` 的列必须是**去重年份计数**（`uniqueN(year)`），
   不能写成 `year_max - year_min + 1`。后者是区间跨度，应另起名如 `span_years`。
   两者放在一起，`n_years < span_years` 直接就是断档判据。
2. 若下游需要知道"缺哪些年"，只存计数还不够，要额外携带年份集合。**用连续区段串
   压缩存储**，而不是逐年布尔向量或逐年行：

   ```r
   # c(1984:1990, 1995, 2003:2017)  ->  "1984-1990;1995;2003-2017"
   # 单年区段只写年份本身，避免 "1995-1995" 这种冗余写法
   subcov_year_segs <- function(yrs) {
     y <- sort(unique(as.integer(yrs))); y <- y[!is.na(y)]
     if (length(y) == 0L) return(NA_character_)
     brk <- c(0L, which(diff(y) > 1L), length(y))     # 相邻年差 > 1 即为切点
     parts <- character(length(brk) - 1L)
     for (i in seq_along(parts)) {
       a <- y[brk[i] + 1L]; b <- y[brk[i + 1L]]
       parts[i] <- if (a == b) as.character(a) else paste0(a, "-", b)
     }
     paste(parts, collapse = ";")
   }
   ```

   选字符串而非 `list` 列的理由：它能直接参与 data.table 的分组聚合、
   跨源 `rbindlist` 与 JSON 序列化，不需要维护 list 列，也不会在 merge 时丢结构。
   代价是取并集时要反解一次（写一个 `segs -> 年份向量 -> segs` 的往返函数即可）。
3. **并集要在正确的粒度上取**。同一国家×作物下的多个变量（面积/产量/单产）
   或同一作物下的多个行政等级，年份覆盖可能不同；直接拿某个子集的区段会低估真实覆盖。
   聚合到展示粒度时必须显式求并集，不能沿用第一行、也不能取 `max(n_years)`。
4. 聚合层级每加深一层，就要问一次"这一层的年份集合是子集的并集，还是随手取的一行"。
   本项目为此在每个数据源入口处单独算一张 `(国家, 作物) -> segs` 表，
   再用 `data.table` 的**更新连接**回填（不用 `merge`，见下条）。

**为什么用更新连接而不是 merge**：`merge` 会重排行，而汇总表一旦错位，
国名就会被安到别的作物上，且不会有任何报错。更新连接保持左表顺序：

```r
catalog[fews_segs, on = .(country_raw = country, crop_en = product),
        year_segs := i.year_segs]
```

两侧列同名时用字符串向量写 `on` 更稳，`.(x = x)` 形式易触发歧义解析：
`on = c(country_raw = "country_name", "crop_en")`。

## 跨源归一化：口径取自来源方文档，不从数据自身推断

把多个来源拼成一张可比目录时，最危险的一类错误是**从数据里反推口径**。
（Hultgren 2025 并入本项目目录时的实测，2026-10-02）

该数据集的 6 个 Stata 面板同时带 `adm1_fact` 与 `uid` 两套行政单元编号。
直觉规则"uid 数明显多于 adm1 数 ⇒ 该国上报到 adm2"对 50 国成立，
但对 **LKA / NIC / VNM 三国不成立**——三国的 uid 是 adm2 粒度编号，论文的上报口径
却是 adm1。按 uid 数反推会让这三国的行政等级虚报深一级，且**不产生任何报错**，
目录里只是凭空多出几个本不存在的下钻层级。

**规则**：

1. 行政层级、统计口径、单位一律以来源方随数据发布的文档（论文附件、README、
   元数据表）为准，数据自身只能作**交叉验证**，不能作判据。本例的做法是：
   层级读自附件表 S1 的"行政单元级别"列，uid 数只用来事后对账。
2. 交叉验证出现不一致时，**先怀疑推断规则**。本例 8 处"不一致"全是推断的假阳性：
   3 国是 uid 粒度导致的误判，5 国（CYP / EST / LTU / LUX / LVA）是只有 1 个
   行政单元、两种层级数量相同而无法区分，只能靠文档定。
3. 文档覆盖的国家多于实际数据时（附件表 54 国、面板实有 53 国，缺 MLT），
   要**显式打印差集**而不是静默丢弃，否则日后核对会误判为数据缺漏。
4. 新来源的作物命名**先查既有键，再决定是否新增归一化规则**。本例中 Hultgren 的
   sorghum / cassava 在既有来源里已是小写键 `sorghum` / `cassava`（覆盖 90 / 50 国）；
   若顺手给它们补 `SORGHUM` / `CASSAVA` 的大写规则，同一作物会裂成两个键，
   界面筛选器里出现两个"高粱"。**判据**：并入后作物键总数不变（本例 767 未变），
   覆盖国家数只增不减。

## 读外部大表：只取必要列，按解析耗时决定是否缓存

`haven::read_dta()` 的 `col_select` 会顺序扫过整个文件但不解析未选中的列。
实测 175 列 / 711 MB / 24.5 万行的面板，只取 6 列耗时 **3.1 秒**、内存 21 MB；
6 个文件共 1.5 GB 合计约 **10 秒**。全列读入既慢又可能爆内存，而其中 169 列
（气象、GDP、时间趋势）与目录登记毫不相干。

```r
# col_select 传外部字符向量时 tidyselect 会发弃用警告，行为正确，静音即可
cols <- c("iso", "year", "area_plant", "area_harv", "production", "yield")
d <- as.data.table(suppressWarnings(read_dta(f, col_select = cols)))
```

配套两条：

- **逐变量判非空再登记**。同一份面板里各国上报的变量并不一致（本例 USA 只报
  播种面积/产量/单产，BRA 四项齐全）。若按"文件里有这一列"登记，会把"该国没报
  这个变量"写成"有数据"。必须先 `keep <- !is.na(d[[v]])` 再分组聚合，
  逐变量单独走一遍。
- **缓存按解析耗时决定，不按文件体积**。FEWS 的 1.7 GB CSV 需要缓存，是因为解析要
  60 秒；Hultgren 的 1.5 GB Stata 读一次只要 10 秒，加缓存反而引入缓存失效风险
  （改了输入却复用旧结果）。两个量级不要用同一套判断。

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
- **Minimise function definitions.** The user explicitly prefers flat, linear scripts. Write a custom function only when it is genuinely reused many times (rule of thumb: four or more call sites, as when the same parser must read four structurally identical files). Two or three similar blocks should stay inline rather than becoming a helper. Never introduce a small one-use helper for tidiness, and never build an "engine + per-country spec" abstraction unless the user asks for it.
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

### Windows: Chinese paths and non-interactive runs

On Windows, an R session launched from a shell or CI runner frequently cannot see a path containing Chinese characters — `file.exists()` and `list.files()` silently return nothing rather than erroring, so the script appears to find no data. The cause is the inherited locale: a shell exporting `LC_ALL=C.UTF-8` leaves R with `LC_CTYPE = "C"`, which cannot translate a UTF-8 path to the native encoding. Set the locale at the top of the script, before any path is used, and print it so the log records which branch ran:

```r
# Windows 下由 bash/CI 启动时，继承的 LC_ALL=C.UTF-8 会把 R 的 LC_CTYPE 降级为 "C"，
# 导致含中文的路径无法被 file.exists()/list.files() 识别，且不报错、只静默返回空。
# 必须在拼接任何路径之前设置，并打印实际值写入日志，便于事后判断走了哪个分支。
if (Sys.info()[["sysname"]] == "Windows") {
  try(Sys.setlocale("LC_CTYPE", "Chinese (Simplified)_China.utf8"), silent = TRUE)
}
print(Sys.getlocale("LC_CTYPE"))
```

A script that must run both interactively in RStudio and non-interactively via `Rscript` should locate its own project root rather than relying on the working directory. Derive it from the script path, with a `getwd()` fallback:

```r
# 脚本自定位项目根目录：RStudio 里可交互运行，Rscript 下也能找到数据，
# 避免把驱动盘绝对路径写死在分析逻辑中
rrs_find_project_dir <- function() {
  args <- commandArgs(trailingOnly = FALSE)
  file_arg <- grep("^--file=", args, value = TRUE)
  script_path <- if (length(file_arg) > 0) {
    sub("^--file=", "", file_arg[1])
  } else if (requireNamespace("rstudioapi", quietly = TRUE) && rstudioapi::isAvailable()) {
    rstudioapi::getSourceEditorContext()$path
  } else {
    NA_character_
  }
  if (is.na(script_path) || script_path == "") return(normalizePath(getwd(), winslash = "/"))
  normalizePath(file.path(dirname(script_path), ".."), winslash = "/")
}
```

The same root cause hits the spatial stack harder. `sf::read_sf()` and `sf::write_sf()` do not degrade silently — they abort with a garbled path, e.g. `Cannot open "C:\...\GYData\..."` where `GYData` has been renamed into mojibake bytes, because GDAL receives the path after it has already been mangled. This fires even for a short **relative** path such as `"Data/Raw_Data/ARG/shp/x.shp"`, since GDAL resolves it against the Chinese working directory. Consequence: a script that reads shapefiles works in RStudio but fails outright under `Rscript` unless the locale is set before the first `sf` call.

Two further Windows traps worth knowing: `unlink("<name>_files", recursive = TRUE)` may also delete the sibling `<name>.html` (observed on R 4.6.x UCRT), so export widget output to `tempdir()` and `file.copy()` it into place; and a function reading `row$col` on a data frame with a renamed column returns a **zero-length** value instead of `NULL`, so the caller silently builds an empty string. Validate required column names at the top of any row-level formatting helper and `stop()` on a missing one.

### Geometry validity must be checked in planar mode

`sf::st_is_valid()` delegates to s2 while `sf_use_s2()` is `TRUE` (the default), so it applies **spherical** rules. A polygon that satisfies the planar GEOS rules can still be reported invalid, and the repair then fails to converge: `st_make_valid()` called with s2 enabled can leave the very same geometries invalid, so a "repair, then verify" step looks broken for no visible reason. For national boundary work, and for any later gridding or intersection, switch to planar first, then repair and re-check:

```r
# 国土边界制图与后续格网切割都按平面处理；s2（球面）校验会把符合平面规则的多边形
# 判为无效，且 s2 开启时 st_make_valid() 修不干净——表现为"修完还是无效"，
# 会让人误以为几何数据本身有问题。
sf::sf_use_s2(FALSE)
bad = which(!sf::st_is_valid(shp))
if (length(bad) > 0) shp[bad, ] = sf::st_make_valid(shp[bad, ])
stopifnot(all(sf::st_is_valid(shp)))
```

Read a boundary file once, report its feature count, geometry types, CRS, and the number of invalid or empty geometries, then write it back and read it again to confirm the round trip. That one block is what makes a map layer reproducible.

### Writing spatial outputs is not atomic

`sf::write_sf()` deletes the existing dataset before writing the new one, so a crash partway through leaves the layer **half destroyed**: the `.shp` is gone while `.dbf`/`.shx`/`.prj` remain at their previous version, and the directory still looks populated. This is not hypothetical — a full script that wrote two layers segfaulted on the second `write_sf()` (exit 139, silent apart from the shell reporting `Segmentation fault`), destroying the `.shp` of a layer that had been written successfully minutes earlier; the same call run on its own, and the whole script on a rerun, both succeeded. Treat it as flaky rather than reproducible, and defend against it:

```r
# write_sf() 先删除旧图层再写新的，中途崩溃会留下半毁状态：
# .shp 已删而 .dbf/.shx/.prj 仍是旧版，目录看着像完整的。
# 故先写到临时位置，校验通过后再整体复制到位——崩溃时旧图层完好无损。
tmp = file.path(tempdir(), "out.shp")
sf::write_sf(layer, tmp)
stopifnot(file.exists(tmp))
file.copy(Sys.glob(sub("\\.shp$", ".*", tmp)), destdir = out_dir, overwrite = TRUE)
```

Suspect this whenever a spatial script dies without an R error message. R processes killed by a signal print nothing; only the calling shell reports the failure, so the log simply stops mid-script.

### Adding a prefix to a key breaks every derivation from that key

Widening a join key (for example prefixing it with an ISO code so it stays unique across countries) silently invalidates code that decodes meaning from the key's position. `substr(idJoin, 1, 2)` to recover Brazilian state codes kept working after `idJoin` became `paste0("BRA", Admin2_ID)` — it just returned `"BR"` for every row, producing a state column of `NA`. No error, no warning. After any change to a key's format, grep for every `substr()`, `nchar()`, `sprintf()` and regex applied to it and re-derive from the original column instead.

### Identifier columns are strings, not numbers

Admin codes and join keys (FIPS, GAUL, IBGE `CD_MUN`, GADM, ADM1/ADM2 ids) must be `character` end to end. `readr::read_csv()` strips quotes and parses a digit-only key as `double`, while `sf::read_sf()` returns the same DBF field as `character`, so the eventual join fails with `Can't join x$idJoin with y$idJoin due to incompatible types` — an error that surfaces at the join, far from its cause. Always pass `col_types = readr::cols(idJoin = readr::col_character())`, or the equivalent per key column, and say why in a comment.

The trap is asymmetric and therefore easy to miss: a key containing a leading zero, such as an Argentine department code like `06441`, stays `character` and works, while the Brazilian 7-digit municipal codes never start with `0` and silently become `double`. Do not conclude from one country that the pattern is safe.

### `pivot_wider()` keeps every unlisted column as an id — leftovers explode the table

`tidyr::pivot_wider()` treats all columns not named in `names_from`/`values_from` as identifiers. A leftover helper column (an intermediate `Value_raw`, a raw code, a per-source label) therefore becomes part of the key, the rows never collapse, and the output silently becomes hundreds of times larger. Two guards:

```r
# 1) 先把只用于中间计算的列显式丢掉，别让它进 pivot_wider 的 id 集合
select(-Value_raw) |>
tidyr::pivot_wider(names_from = Element, values_from = Value)

# 2) 转换后立刻校验：目标键应唯一
stopifnot(!anyDuplicated(df[c("idJoin", "Crop", "Year")]))
```

`values_fn = sum` does **not** protect against this — it only aggregates rows that genuinely share the same key, which these no longer do. A tell-tale sign is an output file orders of magnitude larger than the number of input records (a 782 MB comparison table produced from ~10,000 keys).

### `paste0()` propagates `NA` as the literal string `"NA"`, not as `NA`

`paste0("BRA", NA_character_)` returns `"BRA"` + `"NA"` = `"BRANA"`, a valid-looking primary key that no later `is.na()` will catch. When an administrative code has to be looked up from a name, a failed lookup silently fabricates a fake region. Reorder the pipeline so the filter happens **before** the paste:

```r
# 必须先过滤：paste0("BRA", NA) 会造出 "BRANA"，不会被后面的 is.na() 检出
df |>
  filter(!is.na(code)) |>
  mutate(idJoin = paste0("BRA", code))
```

Say so in a comment at that line — the ordering is load-bearing and looks arbitrary to a later reader.

### `openxlsx` fails on workbooks with non-standard part names

`openxlsx::read.xlsx()` resolves the first worksheet by the conventional part name `xl/worksheets/sheet1.xml`. Some statistics offices' exporters write `xl/worksheets/sheet.xml` with an absolute relationship target; `openxlsx` then returns `character(0)` and the error is `Workbook has no worksheets`, which reads like a corrupt file but is not. Switch engines rather than repairing the file:

```r
# openxlsx 依赖 xl/worksheets/sheet1.xml 这一部件名；该文件用的是 sheet.xml，
# 故改用 readxl，它按关系文件解析，不受部件名限制
raw = readxl::read_excel(path, sheet = 1, col_names = FALSE,
                         col_types = "text", .name_repair = "minimal")
```

Read as `col_types = "text"` when the header rows are irregular (multi-row headers, footnote markers, thousands separators) and parse the columns yourself — guessing types on a table whose first rows are titles produces silent column loss.

### Bare column names only resolve *inside* `[.data.table`

Within `dt[i, j, by]` a column may be named bare (`dt[iso3 == "BRA", .N]`). Outside that bracket the same name is a plain variable lookup in the calling frame, where it does not exist:

```r
# 报错 object 'iso3' not found：这不是 data.table 的子集表达式，而是普通的向量下标，
# iso3 只是 dt 的一列，在全局环境里并不存在
dt$adm[iso3 == iso & dataset_id == ds]

# 正确：把条件留在 i 里
dt[iso3 == iso & dataset_id == ds, max(adm, na.rm = TRUE)]
```

The failure points at whatever line happens to use the pattern, usually far from where the table was built. `dt[!is.na(col)]` / `dt[!is.na(dt$col), ]` are both fine; `vec[col == x]` never is.

Two related traps:

- **Don't reach back into the whole table from inside `j`.** `dt[, .(v = f(dt[iso3 == .BY$iso3, x])), by = iso3]` re-scans the table once per group (O(n²)) and resolves `.BY` through a closure. Compute the per-group aggregate as its own `dt[, .(v = f(x)), by = iso3]` and `merge()` it back.
- **`nzchar(NA)` is `TRUE`.** Emptiness tests on data-derived columns must be `!is.na(x) & nzchar(x)`; `nzchar()` alone treats missing values as present and quietly admits them.

## Review Checklist

When reviewing or refactoring R code, report issues in this order:

1. Reproducibility risks: hard-coded paths, hidden workspace objects, missing packages, encoding problems. On Windows, check that the locale is set before any Chinese path is used — including before the first `sf` call, which errors rather than degrading quietly — and that the script does not depend on the caller's working directory. For spatial outputs, check that each layer is written via a temp location and verified, since `write_sf()` destroys the previous version before writing and a crash leaves the layer half deleted with no R-side error.
2. Data risks: silent row deletion, unclear join keys, identifier columns allowed to be parsed as `double`, unit conversion without comments, overwritten raw data. After any reshaping, check that the intended key is actually unique and that no leftover column was silently kept as a `pivot_wider()` id — the failure mode is a table orders of magnitude too large, and `values_fn` does not prevent it. Check that keys assembled with `paste0()` cannot be built from an `NA` (that yields the literal `"NA"`, not a missing value). When a source publishes multiple distinct missing-value or zero markers, check that each marker has its own explicit mapping rather than a single `as.numeric()` catch-all. When polygons are read or repaired, check that validity is judged in planar mode (`sf_use_s2(FALSE)`) and that the repaired layer is verified, not merely written out. After any change to a join key's format, check every `substr()`/regex derived from it.
3. Statistical risks: unexplained model terms, missing diagnostics, unclear sample definition.
4. Implementation risks: a hand-written routine where an established package exists; an unfamiliar package used without inspecting its source; a replacement that silently changed units, argument order, reference model, missing-value behaviour, or vectorisation.
5. Readability risks: vague names, repeated blocks, very long scripts, scattered output calls.
6. Style issues: `T/F`, inconsistent convention, duplicated `library()`, mixed pipes, commented-out clutter, a one-use helper function where inline code would read better.

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
- How to manage environments, seeds, or reproducibility infrastructure beyond stating that stochastic steps need a seed.

Abstracting repeated code into functions is **not** left open: see the minimisation rule under Readability Rhythm.
