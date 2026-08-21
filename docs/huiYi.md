你的问题可以直接用 QuickSight 的 LAC-A 分层聚合解决，不需要再用 `MAX(分母)`，也不要使用目前 View 中的 `cumulative_nw_connect_count_once`。

## 一、问题原因

当前公式：

```text
sum({monthly_error_count})
/
max({cumulative_nw_connect_count})
```

只在筛选结果唯一确定到：

```text
单月 × 单机种 × 单 Gas Type
```

时正确。

例如 2025/07 全部数据：

| 机种   | Gas Type |     分母 |
| ---- | -------- | -----: |
| PT7  | 13A      | 28,073 |
| PT7  | LPG      |  4,655 |
| PT7+ | 13A      | 18,086 |
| PT7+ | LPG      |  3,056 |

正确总分母应该是：

```text
28,073 + 4,655 + 18,086 + 3,056
= 53,870
```

不能使用 `MAX()` 得到 28,073。

---

## 二、最终解决方案

### 1. 删除 `cumulative_nw_connect_count_once` 的使用

当前 View 通过：

```sql
ROW_NUMBER() OVER (...)
```

选出一条 Error Code 记录保存分母。

这个方式存在严重问题：如果用户筛选 Error Code，恰好把 `row_number = 1` 的记录过滤掉，分母就会变成 0。

因此：

* View 继续保留重复出现的 `cumulative_nw_connect_count`
* QuickSight 通过 LAC-A 按分母真实粒度去重
* 不再使用 `cumulative_nw_connect_count_once`

---

### 2. QuickSight 创建总分母计算字段

字段名：

```text
Denominator_Total
```

计算公式：

```text
sum(
    max(
        {cumulative_nw_connect_count},
        [
            {target_month},
            {series_name},
            {gas_type}
        ]
    )
)
```

这个公式分两步执行：

1. 按 `月份 × 机种 × Gas Type` 取得唯一分母；
2. 将所有剩余组合的唯一分母相加。

QuickSight 支持 `Aggregation(LAC-A())` 形式，例如 `max(sum(...))` 或本方案中的 `sum(max(...))`；而且普通分析筛选器会先应用，再进行 LAC-A 聚合。[AWS LAC-A 官方说明](https://docs.aws.amazon.com/quick/latest/userguide/level-aware-calculations.html)、[QuickSight 计算顺序](https://docs.aws.amazon.com/quick/latest/userguide/order-of-evaluation-quicksight.html)

---

### 3. 创建最终不良率

字段名：

```text
Defect_Rate
```

公式：

```text
ifelse(
    sum(
        max(
            {cumulative_nw_connect_count},
            [
                {target_month},
                {series_name},
                {gas_type}
            ]
        )
    ) = 0,
    null,
    sum({monthly_error_count})
    /
    sum(
        max(
            {cumulative_nw_connect_count},
            [
                {target_month},
                {series_name},
                {gas_type}
            ]
        )
    )
)
```

然后将显示格式设置为百分比。

---

## 三、四个筛选器的结果

使用上述公式后：

| 筛选条件              |                     分母结果 |
| ----------------- | -----------------------: |
| 不筛选               | 所有月份、机种、Gas Type 的唯一分母合计 |
| 机种 = PT7          |    PT7 的所有 Gas Type 分母合计 |
| Gas Type = 13A    |       PT7 13A + PT7+ 13A |
| PT7 + 13A         |                   28,073 |
| Error Code = E001 |                    分母不减少 |
| Error Type = 确定错误 |                    分母不减少 |
| PT7 + E001        |      PT7 的全部 Gas Type 分母 |
| 13A + E001        |             全部机种的 13A 分母 |

以 2025/07 为例：

```text
不筛选：
分母 = 53,870

Gas Type = 13A：
分母 = 28,073 + 18,086
     = 46,159

机种 = PT7：
分母 = 28,073 + 4,655
     = 32,728

PT7 + 13A：
分母 = 28,073
```

这正好覆盖你列出的16种筛选情况。

---

## 四、View 建议修改

删除以下部分：

```sql
ROW_NUMBER() OVER (
  PARTITION BY
    n.target_month,
    n.series_name,
    n.gas_type
  ORDER BY
    d.err_code NULLS LAST,
    d.err_type NULLS LAST
) AS denominator_row_number
```

同时删除：

```sql
CASE
  WHEN g.denominator_row_number = 1
    THEN g.cumulative_nw_connect_count
  ELSE 0
END AS cumulative_nw_connect_count_once
```

保留普通分母：

```sql
g.cumulative_nw_connect_count
```

---

## 五、Error Code、Error Type 完全独立筛选时的改进

你当前的：

```sql
error_dimension AS (
  SELECT DISTINCT
    err_code,
    err_type
  FROM monthly_result
)
```

只生成历史上真实存在过的 Error Code 和 Error Type 组合。

如果两个筛选器允许独立选择，建议改成：

```sql
error_code_dimension AS (
  SELECT DISTINCT err_code
  FROM monthly_result
  WHERE err_code IS NOT NULL
),

error_type_dimension AS (
  SELECT DISTINCT err_type
  FROM monthly_result
  WHERE err_type IS NOT NULL
),

view_grid AS (
  SELECT
    n.target_month,
    n.series_name,
    n.gas_type,
    ec.err_code,
    et.err_type,
    n.cumulative_nw_connect_count
  FROM
    qdx3_fhsbu_tidydata_dev
      .sum_fhsbu_defect_rate_monthly_nw_connect_tidydata n
  CROSS JOIN error_code_dimension ec
  CROSS JOIN error_type_dimension et
)
```

这样即使用户选择了某个没有错误记录的 Error Code + Error Type 组合：

```text
分子 = 0
分母 = 当前月份、机种、Gas Type 的连接台数
不良率 = 0%
```

不会整个结果消失。

---

## 六、告警颜色计算

先创建数值型字段：

```text
Alert_Flag_Number
```

```text
ifelse({is_alert} = true, 1, 0)
```

然后创建：

```text
Current_Filtered_Alert_Flag
```

```text
ifelse(
    sum({monthly_error_count}) > 0
    AND max({Alert_Flag_Number}) = 1,
    1,
    0
)
```

条件格式：

```text
Current_Filtered_Alert_Flag = 1 → 红色
Current_Filtered_Alert_Flag = 0 → 无背景色
```

QuickSight 的表格、透视表和 KPI 都支持根据计算字段设置背景色。[AWS 条件格式官方说明](https://docs.aws.amazon.com/quick/latest/userguide/conditional-formatting-for-visuals.html)

另外，View 中应修正 `alert_type = 3`：

```sql
BOOL_OR(alert_type IN (1, 3)) AS has_new_alert,
BOOL_OR(alert_type IN (2, 3)) AS has_rapid_increase_alert
```

否则原始数据已经是 `alert_type = 3` 时，会被现有的 `alert_type = 1/2` 判断遗漏。

最终最关键的改动就是：

```text
旧分母：
max({cumulative_nw_connect_count})

新分母：
sum(
    max(
        {cumulative_nw_connect_count},
        [{target_month}, {series_name}, {gas_type}]
    )
)
```

这套公式能够正确处理单机种、多机种、单 Gas、多 Gas 和多月份汇总。

需要把原来的“Error Code + Error Type 组合维度”拆成两个独立维度，再分别 `CROSS JOIN`。同时删除 `ROW_NUMBER()` 和 `cumulative_nw_connect_count_once`。

直接将 View 改成下面这样：

```sql
CREATE OR REPLACE VIEW
  qdx3_fhsbu_tidydata_dev.v_sum_fhsbu_visual_defect_rate_monthly_tidydata AS

WITH daily_result AS (

  /* 去除 threshold_group_id 造成的同日重复 */
  SELECT
    judgment_date,
    series_name,
    gas_type,
    err_code,
    err_type,

    MAX(count_judgment) AS daily_count_judgment,
    BOOL_OR(is_alert) AS is_alert,

    /* alert_type = 3 时，代表两种告警都发生 */
    BOOL_OR(alert_type IN (1, 3)) AS has_new_alert,
    BOOL_OR(alert_type IN (2, 3)) AS has_rapid_increase_alert

  FROM
    qdx3_fhsbu_tidydata_dev
      .rec_fhsbu_error_threshold_alert_short_term_tidydata

  GROUP BY
    judgment_date,
    series_name,
    gas_type,
    err_code,
    err_type
),

monthly_result AS (

  /* 按月汇总分子和告警 */
  SELECT
    DATE_TRUNC('month', judgment_date)::date AS target_month,
    series_name,
    gas_type,
    err_code,
    err_type,

    SUM(daily_count_judgment)::bigint AS monthly_error_count,
    BOOL_OR(is_alert) AS is_alert,

    CASE
      WHEN BOOL_OR(has_new_alert)
       AND BOOL_OR(has_rapid_increase_alert)
        THEN 3
      WHEN BOOL_OR(has_new_alert)
        THEN 1
      WHEN BOOL_OR(has_rapid_increase_alert)
        THEN 2
      ELSE NULL
    END AS alert_type

  FROM
    daily_result

  GROUP BY
    DATE_TRUNC('month', judgment_date)::date,
    series_name,
    gas_type,
    err_code,
    err_type
),

/*
 * Error Code 和 Error Type 分别生成独立维度。
 * 不再使用 SELECT DISTINCT err_code, err_type。
 */
error_code_dimension AS (

  SELECT DISTINCT
    err_code

  FROM
    monthly_result

  WHERE
    err_code IS NOT NULL
),

error_type_dimension AS (

  SELECT DISTINCT
    err_type

  FROM
    monthly_result

  WHERE
    err_type IS NOT NULL
),

/*
 * 为每个：
 * 月份 × 机种 × Gas Type
 * 补齐全部 Error Code × Error Type 组合。
 */
view_grid AS (

  SELECT
    n.target_month,
    n.series_name,
    n.gas_type,
    ec.err_code,
    et.err_type,
    n.cumulative_nw_connect_count

  FROM
    qdx3_fhsbu_tidydata_dev
      .sum_fhsbu_defect_rate_monthly_nw_connect_tidydata n

  CROSS JOIN
    error_code_dimension ec

  CROSS JOIN
    error_type_dimension et
)

SELECT
  g.target_month,
  g.series_name,
  g.gas_type,
  g.err_code,
  g.err_type,

  COALESCE(r.monthly_error_count, 0)::bigint
    AS monthly_error_count,

  g.cumulative_nw_connect_count,

  COALESCE(r.is_alert, FALSE)
    AS is_alert,

  r.alert_type

FROM
  view_grid g

LEFT JOIN
  monthly_result r

  ON r.target_month = g.target_month
  AND r.series_name = g.series_name
  AND r.gas_type = g.gas_type
  AND r.err_code = g.err_code
  AND r.err_type = g.err_type;
```

改动重点

原来是：

```sql
error_dimension AS (
  SELECT DISTINCT
    err_code,
    err_type
  FROM monthly_result
)
```

这只能生成实际存在过的组合，例如：

```text
E001 + Error Type 1
E002 + Error Type 2
```

不会生成：

```text
E001 + Error Type 2
```

现在拆成：

```sql
error_code_dimension
error_type_dimension
```

然后：

```sql
CROSS JOIN error_code_dimension
CROSS JOIN error_type_dimension
```

这样就会形成完整组合：

```text
E001 + Type 1
E001 + Type 2
E002 + Type 1
E002 + Type 2
```

如果用户选择一个没有实际错误数据的组合：

```text
Error Code = E001
Error Type = Type 2
```

结果为：

```text
monthly_error_count = 0
is_alert = false
分母仍然存在
不良率 = 0
```

QuickSight 分母公式继续使用：

```text
sum(
    max(
        {cumulative_nw_connect_count},
        [
            {target_month},
            {series_name},
            {gas_type}
        ]
    )
)
```

不要使用：

```text
sum({cumulative_nw_connect_count})
```

也不要再使用：

```text
cumulative_nw_connect_count_once
```

需要注意：拆分后 View 行数约为：

```text
分母组合数 × Error Code 数量 × Error Type 数量
```

如果 Error Code 数量特别多，View/SPICE 数据量会明显增大；但从逻辑正确性来说，这是支持两个筛选器完全独立选择的直接方案。


