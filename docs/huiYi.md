Jo，你提出的这两个问题都很有道理。尤其是 `DATE_TRUNC()`，理解它为什么出现在 JOIN 条件中，就能理解整个数据合并的设计。

另外，`record_source` 可以删除。 如果你希望通过 `alert_type = 4` 识别长期告警，那么我们可以让 long_term 独有记录的 `alert_type` 直接等于 4。

不过这里有一个小前提：short_term 原本的 `alert_type` 不能也取值为 4，否则仅凭这个字段无法区分记录来源。

# 一、为什么要使用 DATE_TRUNC('month', judgment_date)？

先看一个具体例子。

假设两张表中分别存在以下记录：

short_term

<table class="_6IUVGW_Table" data-d-column-sizing="auto" data-d-dividers="" style="table-layout: auto;"><tbody><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">judgment_date</td><td data-d-component="table-cell" data-d-valign="start">err_code</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">count_judgment</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">2025-05-01</td><td data-d-component="table-cell" data-d-valign="start">78F2</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">10</td></tr></tbody></table>

long_term

<table class="_6IUVGW_Table" data-d-column-sizing="auto" data-d-dividers="" style="table-layout: auto;"><tbody><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">judgment_date</td><td data-d-component="table-cell" data-d-valign="start">err_code</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">alert_type</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">2025-05-05</td><td data-d-component="table-cell" data-d-valign="start">78F2</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">4</td></tr></tbody></table>

如果直接按照日期进行关联：

SQL

```
ON s.judgment_date = l.judgment_date
```

那么：

```
2025-05-01 = 2025-05-05

结果：FALSE
```

两条记录无法匹配。

但是，你的 QuickSight Pivot Table 是按照月份聚合的。

也就是说，在业务上，这两条记录都属于：

2025 年 5 月的 78F2。

因此，我们需要在关联时忽略日期中的具体日，只比较年月。

## 1. DATE_TRUNC 的具体作用

PostgreSQL 的 `DATE_TRUNC()` 用于将日期或时间截断到指定的时间单位。

例如：

SQL

```
SELECT
    DATE_TRUNC('month', DATE '2025-05-05');
```

结果是：

```
2025-05-01 00:00:00
```

它不是把日期转换为字符串 `2025-05`，而是将时间截断到该月的第一天。

下面可以看到不同日期经过处理后的结果：

|
原始日期

|

DATE_TRUNC('month', ...)

|
| --- | --- |
|

2025-05-01

|

2025-05-01

|
|

2025-05-05

|

2025-05-01

|
|

2025-05-28

|

2025-05-01

|
|

2025-06-01

|

2025-06-01

|

这里省略了结果中的时间部分。

因此：

SQL

```
DATE_TRUNC('month', s.judgment_date)
=
DATE_TRUNC('month', l.judgment_date)
```

实际上是在判断：

两条记录是不是属于同一个月份？

## 2. 为什么后面还要加 ::date？

你之前看到的是：

SQL

```
DATE_TRUNC('month', s.judgment_date)::date
```

其中的 `::date` 是 PostgreSQL 的类型转换语法。

它将截断后的时间值转换成 DATE 类型。

例如：

```
DATE_TRUNC 的结果：
2025-05-01 00:00:00

转换为 date 后：
2025-05-01
```

对于你当前按月关联的需求，这种写法比较直观。

不过，`::date` 并不是实现月份匹配的必要条件。如果两侧使用兼容的时间类型，直接比较 `DATE_TRUNC()` 的结果也可以。

# 二、为什么 SQL 要按月份关联，而不是由 QuickSight 处理？

这两个操作虽然都涉及月份，但作用完全不同。

PostgreSQL

DATE_TRUNC('month', judgment_date)

用于判断两张表中的记录是否属于同一个月份，从而确定应该把哪个长期告警关联到哪个短期数据。

QuickSight

judgment_date → 月

用于将关联后的每日明细按月份分组，再计算 SUM(count_judgment)。

因此，虽然我们在 SQL 中使用了 `DATE_TRUNC()`，但是：

* short_term 原始的 `judgment_date` 不会被修改。

* short_term 原始的 `count_judgment` 不会被修改。

* QuickSight 依然按照原来的方式按月聚合。

这里的 `DATE_TRUNC()` 只是用于匹配长期告警，不是为了提前计算 Pivot 的月度件数。

# 三、删除 record_source 后的最终 SQL

按照你的要求，这次进行以下修改：

* 删除 `record_source`。

* 保留 `is_alert_long`，长期告警时值为 4。

* long_term 独有记录的 `alert_type` 也设置为 4。

* long_term 独有记录的 `count_judgment` 设置为 0。

* short_term 的原始明细和件数保持不变。

SQL

```
WITH long_alert AS (

    -- 1. 每个月、每个业务组合保留一条长期告警

    SELECT DISTINCT ON (
        DATE_TRUNC('month', judgment_date)::date,
        series_name,
        gas_type,
        err_code,
        err_type
    )

        judgment_date,
        series_name,
        gas_type,
        err_code,
        err_type,

        threshold_group_id,
        threshold_group_name,

        4 AS is_alert_long

    FROM
        qdx3_fhsbu_tidydata_dev
        .rec_fhsbu_error_threshold_alert_long_term_tidydata

    WHERE alert_type = 4

    ORDER BY
        DATE_TRUNC('month', judgment_date)::date,
        series_name,
        gas_type,
        err_code,
        err_type,
        judgment_date
),

short_data AS (

    -- 2. 保留 short_term 的所有原始记录

    SELECT
        s.judgment_date,
        s.series_name,
        s.gas_type,
        s.err_code,
        s.err_type,

        s.threshold_group_id,
        s.threshold_group_name,

        s.threshold_a,
        s.threshold_b,
        s.threshold_c,
        s.threshold_d,

        s.alert_type,
        s.count_judgment,
        s.count_comparison_avg,
        s.is_alert,

        COALESCE(
            l.is_alert_long,
            0
        ) AS is_alert_long,

        CASE
            WHEN s.alert_type > 0
              OR COALESCE(l.is_alert_long, 0) = 4
            THEN 1
            ELSE 0
        END AS alert_color_flag

    FROM
        qdx3_fhsbu_tidydata_dev
        .rec_fhsbu_error_threshold_alert_short_term_tidydata s

    LEFT JOIN long_alert l

        ON DATE_TRUNC('month', s.judgment_date)::date
           = DATE_TRUNC('month', l.judgment_date)::date

        AND s.series_name = l.series_name
        AND s.gas_type = l.gas_type
        AND s.err_code = l.err_code
        AND s.err_type = l.err_type
),

long_only_data AS (

    -- 3. 补充 short_term 中不存在的月度长期告警

    SELECT
        l.judgment_date,
        l.series_name,
        l.gas_type,
        l.err_code,
        l.err_type,

        l.threshold_group_id,
        l.threshold_group_name,

        NULL AS threshold_a,
        NULL AS threshold_b,
        NULL AS threshold_c,
        NULL AS threshold_d,

        -- 按你的要求，长期独有记录的 alert_type = 4
        4 AS alert_type,

        -- 不影响短期件数统计
        0 AS count_judgment,

        NULL AS count_comparison_avg,
        NULL AS is_alert,

        -- 长期告警标志
        4 AS is_alert_long,

        -- 红色背景
        1 AS alert_color_flag

    FROM long_alert l

    WHERE NOT EXISTS (

        SELECT 1

        FROM
            qdx3_fhsbu_tidydata_dev
            .rec_fhsbu_error_threshold_alert_short_term_tidydata s

        WHERE
            DATE_TRUNC('month', s.judgment_date)::date
            = DATE_TRUNC('month', l.judgment_date)::date

            AND s.series_name = l.series_name
            AND s.gas_type = l.gas_type
            AND s.err_code = l.err_code
            AND s.err_type = l.err_type
    )
)

-- 4. 合并最终数据

SELECT *
FROM short_data

UNION ALL

SELECT *
FROM long_only_data;
```

这份 SQL 沿用上一版的字段清单；如果实际表字段的数据类型或名称与之前的截图不一致，需要以数据库实际定义为准。另外，以上关联假定五个业务维度的字段不为 NULL；如果允许 NULL 且需要将两侧 NULL 视为相同，应将对应的 `=` 改为 `IS NOT DISTINCT FROM`。

# 四、修改后，78F2 的数据会是什么样？

对于你发现的 2025 年 5 月 5 日的长期告警，如果 short_term 在同月、同业务组合下完全没有记录，那么 SQL 将生成：

|
字段

|

值

|
| --- | --- |
|

judgment_date

|

2025-05-05

|
|

err_code

|

78F2

|
|

alert_type

|

4

|
|

count_judgment

|

0

|
|

is_alert_long

|

4

|
|

alert_color_flag

|

1

|

QuickSight 最终可以显示：

## 発生件数

<table class="_6IUVGW_Table" data-d-column-sizing="auto" data-d-dividers="" style="table-layout: auto;"><tbody><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-has-width="" data-d-valign="start" style="width: 50%;">単月件数</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start">78F2</td></tr><tr data-d-component="table-row"><td data-d-component="table-cell" data-d-valign="start">2025年05月</td><td data-d-align="end" data-d-component="table-cell" data-d-valign="start"><div class="oIb9lq_Box" data-d-auto-spacing="" data-d-component="box" data-d-direction="col" style="--smoothing-background-color: #c62828; border-radius: var(--radius-xs); --w-box-gutter-block-end: calc(var(--spacing, 0.25rem) * 2); --w-box-gutter-block-start: calc(var(--spacing, 0.25rem) * 2); --w-box-gutter-inline-end: calc(var(--spacing, 0.25rem) * 2); --w-box-gutter-inline-start: calc(var(--spacing, 0.25rem) * 2); padding-block: calc(var(--spacing, 0.25rem) * 2); padding-inline: calc(var(--spacing, 0.25rem) * 2); background-color: rgb(198, 40, 40);"><h2 class="w6asjq_TextBase GgxHUa_Title" data-d-component="title" data-d-size="lg" data-d-weight="semibold" data-d-text-align="end" style="color: rgb(255, 255, 255);">0</h2></div></td></tr></tbody></table>

QuickSight 的配置仍然保持：

|
项目

|

设置

|
| --- | --- |
|

Rows

|

judgment_date（月）

|
|

Columns

|

err_code

|
|

Values

|

SUM(count_judgment)

|
|

条件格式

|

MAX(alert_color_flag) > 0

|
|

背景色

|

红色

|

最后补充一个字段语义上的区别：`alert_type = 4` 可以用来标记我们人工补充的长期独有记录，但对于同时存在短期和长期告警的记录，`alert_type` 仍然保留 short_term 原值。因此，如果你要判断某条数据是否存在长期告警，统一使用 `is_alert_long = 4` 更准确。



-----sql修改

```sql

WITH long_alert AS (

    -- 1. 每个月、每个业务组合保留一条长期告警

    SELECT DISTINCT ON (
        DATE_TRUNC('month', judgment_date)::date,
        series_name,
        gas_type,
        err_code,
        err_type
    )

        judgment_date,
        series_name,
        gas_type,
        err_code,
        err_type,

        threshold_group_id,
        threshold_group_name,

        4::numeric AS alert_type_long

    FROM
        qdx3_fhsbu_tidydata_dev
        .rec_fhsbu_error_threshold_alert_long_term_tidydata

    WHERE alert_type = 4

    ORDER BY
        DATE_TRUNC('month', judgment_date)::date,
        series_name,
        gas_type,
        err_code,
        err_type,
        judgment_date
),

short_data AS (

    -- 2. 保留 short_term 的全部原始记录

    SELECT
        s.judgment_date,
        s.series_name,
        s.gas_type,
        s.err_code,
        s.err_type,

        s.threshold_group_id,
        s.threshold_group_name,

        s.threshold_a,
        s.threshold_b,
        s.threshold_c,
        s.threshold_d,

        s.alert_type,
        s.count_judgment,
        s.count_comparison_avg,
        s.is_alert,

        -- 长期告警类型：4 或 0
        COALESCE(
            l.alert_type_long,
            0
        ) AS alert_type_long,

        -- 综合背景色判断：1 或 0
        CASE
            WHEN s.alert_type > 0
              OR COALESCE(l.alert_type_long, 0) = 4
            THEN 1
            ELSE 0
        END AS alert_color_flag

    FROM
        qdx3_fhsbu_tidydata_dev
        .rec_fhsbu_error_threshold_alert_short_term_tidydata s

    LEFT JOIN long_alert l

        ON DATE_TRUNC('month', s.judgment_date)::date
           = DATE_TRUNC('month', l.judgment_date)::date

        AND s.series_name = l.series_name
        AND s.gas_type = l.gas_type
        AND s.err_code = l.err_code
        AND s.err_type = l.err_type
),

long_only_data AS (

    -- 3. 补充 short_term 中不存在的月度长期告警

    SELECT
        l.judgment_date,
        l.series_name,
        l.gas_type,
        l.err_code,
        l.err_type,

        l.threshold_group_id,
        l.threshold_group_name,

        NULL::numeric AS threshold_a,
        NULL::numeric AS threshold_b,
        NULL::numeric AS threshold_c,
        NULL::numeric AS threshold_d,

        -- 长期独有记录的 alert_type = 4
        4::numeric AS alert_type,

        -- 不影响短期件数统计
        0::numeric AS count_judgment,

        NULL::numeric AS count_comparison_avg,
        NULL::boolean AS is_alert,

        -- 长期告警类型
        4::numeric AS alert_type_long,

        -- 长期告警独有记录直接标记为红色
        1 AS alert_color_flag

    FROM long_alert l

    WHERE NOT EXISTS (

        SELECT 1

        FROM
            qdx3_fhsbu_tidydata_dev
            .rec_fhsbu_error_threshold_alert_short_term_tidydata s

        WHERE
            DATE_TRUNC('month', s.judgment_date)::date
            = DATE_TRUNC('month', l.judgment_date)::date

            AND s.series_name = l.series_name
            AND s.gas_type = l.gas_type
            AND s.err_code = l.err_code
            AND s.err_type = l.err_type
    )
)

-- 4. 合并最终数据

SELECT *
FROM short_data

UNION ALL

SELECT *
FROM long_only_data;
```
