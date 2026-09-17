Jo，按照你最终确认的规则，现在可以确定 SQL 的合并方式了。

最终方案：保留 short_term 的全部原始明细，将 long_term 按「月份＋机种＋气体种类＋错误代码＋错误分类」汇总为警报标志，再进行 LEFT JOIN。

这样可以同时满足三个要求：

* `short_term.count_judgment` 的原始值不变，记录不增加、不减少。

* QuickSight 继续使用 `judgment_date（月）` 和 `SUM(count_judgment)`。

* 同月、同机种、同气体种类、同错误代码、同错误分类下，只要短期警报大于 0，或者长期警报等于 4，对应的 Pivot 单元格就变红。

不过有一点需要区分：你的 Pivot 目前只显示月份和错误代码，因此**多个机种或气体种类汇总到同一个单元格时，只要其中一个符合警报条件，该单元格就会变红。**如果用户通过筛选器选择某个机种，颜色判断则应只针对筛选后保留的数据。

# 一、最终 PostgreSQL SQL

下面的 SQL 使用你截图中的实际表名，可以作为 QuickSight 的 Custom SQL。

SQL

```
WITH long_alert AS (

    -- 1. 按月份和五个业务维度汇总长期警报
    SELECT
        DATE_TRUNC('month', judgment_date)::date
            AS alert_month,

        series_name,
        gas_type,
        err_code,
        err_type,

        -- 只判断是否存在 alert_type = 4
        MAX(
            CASE
                WHEN alert_type = 4 THEN 1
                ELSE 0
            END
        ) AS long_alert_flag

    FROM
        qdx3_fhsbu_tidydata_dev
        .rec_fhsbu_error_threshold_alert_long_term_tidydata

    GROUP BY
        DATE_TRUNC('month', judgment_date)::date,
        series_name,
        gas_type,
        err_code,
        err_type
)

SELECT

    -- 2. 保留 short_term 全部原始字段
    s.*,

    -- 3. 长期警报标志
    COALESCE(
        l.long_alert_flag,
        0
    ) AS long_alert_flag,

    -- 4. 最终背景色判断字段
    CASE
        WHEN s.alert_type > 0
          OR COALESCE(l.long_alert_flag, 0) = 1
        THEN 1
        ELSE 0
    END AS alert_color_flag

FROM
    qdx3_fhsbu_tidydata_dev
    .rec_fhsbu_error_threshold_alert_short_term_tidydata AS s

LEFT JOIN long_alert AS l

    -- 月份一致
    ON DATE_TRUNC('month', s.judgment_date)::date
       = l.alert_month

    -- 机种一致
    AND s.series_name = l.series_name

    -- 气体种类一致
    AND s.gas_type = l.gas_type

    -- 错误代码一致
    AND s.err_code = l.err_code

    -- 错误分类一致
    AND s.err_type = l.err_type;
```

注意：这里使用的是 `LEFT JOIN`，而不是 `INNER JOIN`。即使 long_term 没有对应记录，也必须保留 short_term 的原始数据。

# 二、用实际数据理解 SQL 的执行结果

假设 short_term 有以下四条记录：

### short_term（原始明细）

<table class="_6IUVGW_Table" data-d-column-sizing="auto" data-d-dividers="" style="table-layout: auto;"><tbody><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">日期</td><td data-d-component="table-cell" data-d-valign="start">机种</td><td data-d-component="table-cell" data-d-valign="start">错误代码</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">件数</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">08-01</td><td data-d-component="table-cell" data-d-valign="start">PT7</td><td data-d-component="table-cell" data-d-valign="start">A8F0</td><td data-d-component="table-cell" data-d-valign="start">10</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">08-02</td><td data-d-component="table-cell" data-d-valign="start">PT7</td><td data-d-component="table-cell" data-d-valign="start">A8F0</td><td data-d-component="table-cell" data-d-valign="start">20</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">08-03</td><td data-d-component="table-cell" data-d-valign="start">PT7</td><td data-d-component="table-cell" data-d-valign="start">A8F0</td><td data-d-component="table-cell" data-d-valign="start">30</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">08-04</td><td data-d-component="table-cell" data-d-valign="start">PT7+</td><td data-d-component="table-cell" data-d-valign="start">A8F0</td><td data-d-component="table-cell" data-d-valign="start">40</td></tr></tbody></table>

示例假设各行的 gas_type 和 err_type 相同，短期警报均不触发。

long_term 中有一条记录：

### long_term（长期警报）

<table class="_6IUVGW_Table" data-d-column-sizing="auto" data-d-dividers="" style="table-layout: auto;"><tbody><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">日期</td><td data-d-component="table-cell" data-d-valign="start">机种</td><td data-d-component="table-cell" data-d-valign="start">错误代码</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">alert_type</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">08-05</td><td data-d-component="table-cell" data-d-valign="start">PT7</td><td data-d-component="table-cell" data-d-valign="start">A8F0</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start"><div class="ANObbW_Badge lKEGNW_Badge" data-color="danger" data-size="sm" data-pill="" data-variant="soft" data-d-component="badge" data-d-weight="medium">4</div></td></tr></tbody></table>

执行 SQL 后：

|
日期

|

机种

|

count_judgment

|

alert_color_flag

|
| --- | --- | --- | --- |
|

08-01

|

PT7

|

10

|

1

|
|

08-02

|

PT7

|

20

|

1

|
|

08-03

|

PT7

|

30

|

1

|
|

08-04

|

PT7+

|

40

|

0

|

可以发现：

* PT7 的三条数据都匹配到了同月的长期警报。

* PT7+ 因为机种不同，不会匹配 PT7 的长期警报。

* 四条 short_term 记录全部保留。

* `count_judgment` 的数值完全不变。

当 QuickSight 不筛选机种时，这四条记录将汇总到同一个月份、同一个错误代码的单元格。

### QuickSight 最终显示效果

<table class="_6IUVGW_Table" data-d-column-sizing="auto" data-d-dividers="" style="table-layout: auto;"><tbody><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">単月件数</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">A8F0</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">2026年08月</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start"><div class="oIb9lq_Box" data-d-auto-spacing="" data-d-component="box" data-d-direction="col" style="--smoothing-background-color: #c62828; border-radius: var(--radius-xs); --w-box-gutter-block-end: calc(var(--spacing, 0.25rem) * 2); --w-box-gutter-block-start: calc(var(--spacing, 0.25rem) * 2); --w-box-gutter-inline-end: calc(var(--spacing, 0.25rem) * 2); --w-box-gutter-inline-start: calc(var(--spacing, 0.25rem) * 2); padding-block: calc(var(--spacing, 0.25rem) * 2); padding-inline: calc(var(--spacing, 0.25rem) * 2); background-color: rgb(198, 40, 40);"><h2 class="w6asjq_TextBase GgxHUa_Title" data-d-component="title" data-d-size="lg" data-d-weight="semibold" data-d-text-align="end" style="color: rgb(255, 255, 255);">100</h2></div></td></tr></tbody></table>

SUM(count_judgment) = 100；MAX(alert_color_flag) = 1，因此显示红色。

如果在 Dashboard 中筛选 `series_name = PT7+`，那么只剩下 40 这一条数据，颜色也会恢复正常。

这正是我们在 JOIN 中加入机种、气体种类和错误分类的意义。

# 三、QuickSight 的具体修改方法

你当前的 Pivot 配置无需重新制作。

## 保留现有 Pivot 配置

行（Rows）

judgment_date

集計：月

不修改

列（Columns）

err_code

不修改

值（Values）

count_judgment

合計（SUM）

不修改

只需要更新数据集，并调整背景色条件。

## 背景色条件设置

将原先基于 `alert_type` 的判断替换为以下配置：

|
设置项目

|

设置值

|
| --- | --- |
|

格式化目标

|

count_judgment

|
|

判断依据

|

alert_color_flag

|
|

聚合方式

|

MAX

|
|

条件

|

大于 0

|
|

背景色

|

红色

|

这样，QuickSight 在计算月份件数时仍然使用 SUM，而在判断背景颜色时使用 MAX。

如果当前条件格式已经存在，建议直接修改原来的规则，避免新旧两条规则同时生效，造成颜色冲突。

# 四、验证 SQL 是否真的没有影响件数

这是正式替换 QuickSight 数据集前必须检查的一步。

下面提供一条可以直接运行的验证 SQL。

将第一部分的完整 SQL 放进 `merged_data` 中：

SQL

```
WITH long_alert AS (
    -- 第一部分的 long_alert 查询
),
merged_data AS (
    -- 第一部分的最终 SELECT 查询
)
SELECT
    (SELECT COUNT(*)
     FROM qdx3_fhsbu_tidydata_dev
         .rec_fhsbu_error_threshold_alert_short_term_tidydata)
         AS original_rows,

    (SELECT COUNT(*)
     FROM merged_data)
         AS merged_rows,

    (SELECT SUM(count_judgment)
     FROM qdx3_fhsbu_tidydata_dev
         .rec_fhsbu_error_threshold_alert_short_term_tidydata)
         AS original_count,

    (SELECT SUM(count_judgment)
     FROM merged_data)
         AS merged_count;
```

这里是验证 SQL 的结构示例，两个注释位置需要替换为第一部分对应的实际查询，不能直接将注释当作完整 SQL 执行。

预期结果：

|
original_rows

|

merged_rows

|

original_count

|

merged_count

|
| --- | --- | --- | --- |
|

1000

|

1000

|

5000

|

5000

|

以上数字仅为示例。实际执行时，两组结果必须分别相等。

更严格的验证是比较每个「月份＋错误代码」的 SUM，确保 Pivot 每个单元格的件数也一致。

# 五、一个容易忽略的情况：长期警报只有 long_term 有记录

假设：

```
short_term：
2026-08 | A8F0 | 存在数据

long_term：
2026-08 | A9F0 | alert_type = 4
```

如果 short_term 中完全没有 A9F0 的数据，那么采用上述 LEFT JOIN 后，Pivot 不会自动生成一个新的 A9F0 单元格。

因为我们是以 short_term 为主表。

这符合「不改变原来的统计件数」的设计，但如果客户还要求显示仅存在于 long_term 的错误代码，就需要另外设计数据集，不能简单地把 long_term 的记录直接追加进去。

最终结论：使用第一部分的 SQL，保留 short_term 原始明细，对 long_term 按月份和全部业务维度进行警报汇总，再 LEFT JOIN。

这是在你已确认的匹配规则下，既能保持 `SUM(count_judgment)` 不变，又能让长期警报参与 QuickSight 背景色判断的实现方式。
