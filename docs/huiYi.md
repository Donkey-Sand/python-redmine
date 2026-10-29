可以。下面给你一套**从 DB 到 QuickSight，按顺序操作即可的实施方案**。

最终效果固定为：**页面只有一个多选下拉框，选择 AA、BB 后，只显示同一条记录中同时包含 AA 和 BB 的数据。**

下面按 **PostgreSQL** 编写。由于我没有连接你的实际 DB 和 QS，真实表名需要你替换；每个阶段都安排了检查步骤。

---

**第 0 步：先确认名称和实施范围**

本次使用以下名称，建议新建对象时直接沿用，避免配置混淆。

| 对象 | 本文使用的名称 | 作用 |
|---|---|---|
| 现有错误明细表或视图 | `public.your_error_source` | **替换成你的真实来源** |
| 新建明细视图 | `public.v_qs_error_and_detail` | 增加拼接字段 |
| 新建候选视图 | `public.v_qs_error_options` | 提供独立错误选项 |
| QS 明细数据集 | `Warranty_Error_AND_DQ` | 根据选择查询明细 |
| QS 候选数据集 | `Warranty_Error_Options` | 提供下拉选项 |
| 数据集参数 | `SelectedErrors` | SQL 接收的参数 |
| 分析参数 | `pSelectedErrors` | 页面控件使用的参数 |

四个错误字段暂按以下名称：

```text
work_equip_fail_dp_w_01
work_equip_fail_dp_w_02
work_equip_fail_dp_w_03
work_equip_fail_dp_w_04
```

开始前确认三件事：

- 来源数据是一条记录里保存四个错误字段，符合你要判断的记录粒度。
- 错误内容本身不含 `|`，也不存在名为 `指定なし` 的真实错误。
- **先使用纯错误明细验证。**如果现有数据集混有网络连接台数等分母记录，先不要直接替换它，后面第 11 步单独处理。

以下示例在 `public` 下创建新视图；如果公司要求使用其他 schema，将新视图的 `public` 统一替换。

---

**第 1 步【DB】：建立带 `error_all` 的明细视图**

在你平时使用的数据库客户端打开 SQL 编辑器，连接到 QS 使用的 PostgreSQL 数据库。

执行下面的 SQL，**把 `public.your_error_source` 替换为真实表名或视图名**：

```sql
CREATE VIEW public.v_qs_error_and_detail AS
SELECT
    t.*,
    concat_ws(
        '|',
        NULLIF(t.work_equip_fail_dp_w_01::text, ''),
        NULLIF(t.work_equip_fail_dp_w_02::text, ''),
        NULLIF(t.work_equip_fail_dp_w_03::text, ''),
        NULLIF(t.work_equip_fail_dp_w_04::text, '')
    ) AS error_all
FROM public.your_error_source AS t;
```

这条 SQL 的作用：

- 保留原来的字段。
- 增加一个 `error_all` 字段。
- 把非空错误用 `|` 连接。
- NULL 和空字符串不参与连接。

例如：

| 错误① | 错误② | 错误③ | 错误④ | error_all |
|---|---|---|---|---|
| AA | BB | NULL | CC | `AA|BB|CC` |
| BB | NULL | AA | NULL | `BB|AA` |
| AA | 空字符串 | NULL | NULL | `AA` |

`concat_ws` 会跳过 NULL；`NULLIF(..., '')` 用于把空字符串转成 NULL。[www.postgresql.org](https://www.postgresql.org/docs/current/functions-string.html?utm_source=chatgpt.com)

**完成后检查：**

```sql
SELECT
    work_equip_fail_dp_w_01,
    work_equip_fail_dp_w_02,
    work_equip_fail_dp_w_03,
    work_equip_fail_dp_w_04,
    error_all
FROM public.v_qs_error_and_detail
LIMIT 30;
```

确认 `error_all` 与四个原字段对应正确。

再检查行数：

```sql
SELECT
    (SELECT COUNT(*) FROM public.your_error_source) AS source_rows,
    (SELECT COUNT(*) FROM public.v_qs_error_and_detail) AS view_rows;
```

**预期：两个行数相同。**新增拼接字段不应增加或减少记录。

> 如果提示没有创建视图的权限，让负责 DB 的人员执行这段 SQL 即可。若同名视图已存在，先确认其用途，或为本次测试换一个新名称。

---

**第 2 步【DB】：建立下拉框候选视图**

这个视图只需要返回一列：

```text
error_code
```

其中每行是一个独立错误，另外加一个“不限制”的选项 `指定なし`。

执行：

```sql
CREATE VIEW public.v_qs_error_options AS
SELECT DISTINCT
    e.error_code
FROM public.v_qs_error_and_detail AS t
CROSS JOIN LATERAL
    unnest(string_to_array(t.error_all, '|')) AS e(error_code)
WHERE e.error_code IS NOT NULL
  AND e.error_code <> ''

UNION

SELECT '指定なし'::text AS error_code;
```

这段 SQL 会把：

```text
AA|BB|CC
BB|AA
AA|DD
```

整理成：

```text
AA
BB
CC
DD
指定なし
```

**完成后检查：**

```sql
SELECT error_code
FROM public.v_qs_error_options
ORDER BY error_code;
```

预期：

- 每行只有一个错误。
- 没有重复错误。
- 没有空选项。
- 有且只有一个 `指定なし`。

**这份候选列表只用于下拉框，不要 JOIN 到明细数据集。**这样不会造成明细重复。

---

**第 3 步【DB】：先验证 AND 判断**

在连接 QS 之前，先用两个真实错误测试数据库逻辑。

下面的 `AA`、`BB` 要替换成实际存在的两个错误值，最好先选一条已知同时包含这两个错误的记录。

```sql
SELECT
    t.*
FROM public.v_qs_error_and_detail AS t
WHERE
    array_remove(
        string_to_array(t.error_all, '|'),
        ''
    ) @> ARRAY['AA', 'BB']::text[]
LIMIT 100;
```

这里：

```sql
@> ARRAY['AA', 'BB']::text[]
```

表示“必须同时包含 AA 和 BB”，不要求顺序相同或相邻。[docs.aws.amazon.com](https://www.postgresql.org/docs/current/functions-array.html?utm_source=chatgpt.com)

**检查预期：**

| error_all | 是否应该返回 |
|---|---|
| `AA|BB|CC` | 是 |
| `BB|CC|AA` | 是 |
| `AA|CC` | 否 |
| `AA|AA|CC` | 否 |
| `BB|DD` | 否 |

如果没有返回数据，先确认实际数据中是否确实存在这个组合。

**这一步通过，再继续 QS 配置。**

---

**第 4 步【QS】：创建候选数据集**

打开 QuickSight，进入数据集页面。不同界面的文字可能略有差异，下面同时提供英文功能名称。

操作顺序：

1. 进入 **Datasets／データセット**。
2. 点击 **New dataset／新しいデータセット**。
3. 选择现有的 PostgreSQL 数据源。
4. 选择 **Use custom SQL／カスタム SQL を使用**。
5. 输入下面的 SQL：

```sql
SELECT error_code
FROM public.v_qs_error_options
```

6. 数据集名称填写：

```text
Warranty_Error_Options
```

7. 选择 **SPICE**。
8. 保存并完成导入。

**检查结果：**

数据集中只有一个 String 字段：

```text
error_code
```

预览能看到独立错误值和 `指定なし`。

这里使用 SPICE 是为了让候选列表读取方便。如果后续出现新错误，需要刷新这份候选数据集；可按业务安排每日刷新。

AWS 官方也采用独立候选数据集来提供完整选项，避免列表随着明细筛选而缩小。[Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-advanced-options.html?utm_source=chatgpt.com)

---

**第 5 步【QS】：创建 Direct Query 明细数据集**

再次新建数据集，选择同一个 PostgreSQL 数据源。

先使用下面这条**不带参数**的 SQL：

```sql
SELECT t.*
FROM public.v_qs_error_and_detail AS t
```

数据集名称填写：

```text
Warranty_Error_AND_DQ
```

**查询模式选择 Direct Query／直接查询。**

进入数据编辑／预览页面，确认可以看到：

- 四个原始错误字段；
- `error_all`；
- 记录 ID 等原有字段。

如需先保存才能继续编辑，就保存后重新进入该数据集的编辑页面。

**这一阶段预览应能显示未按错误筛选的明细。**

这里必须使用 Direct Query，因为后续要让用户选择传入自定义 SQL。SPICE 不支持这种随选择重新执行的数据集参数 SQL。[docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters.html?utm_source=chatgpt.com)

---

**第 6 步【QS】：创建数据集参数**

现在仍然在 `Warranty_Error_AND_DQ` 的**数据集编辑页面**。

1. 打开左侧 **Parameters／パラメータ**。
2. 点击 `＋`。
3. 按下表填写：

| 配置项 | 填写内容 |
|---|---|
| Name | `SelectedErrors` |
| Data type | `String` |
| Values | **Multiple values** |
| Default value | `指定なし` |

4. 点击 **Create**。

注意：

- UI 中填写默认值时，直接输入 `指定なし`，**不加单引号**。
- 必须选择 **Multiple values／複数の値**。
- 参数名与后面的 SQL 保持完全一致。

AWS 官方说明，单值／多值属性创建后不能直接修改，因此这里不要误选 Single value。[Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-SQL.html?utm_source=chatgpt.com)

---

**第 7 步【QS】：把参数加入明细 SQL**

在同一个数据集中，打开自定义 SQL 的编辑功能，将第 5 步的基础 SQL 替换为：

```sql
SELECT t.*
FROM public.v_qs_error_and_detail AS t
WHERE
    (
        '指定なし' IN (<<$SelectedErrors>>)

        OR

        array_remove(
            string_to_array(t.error_all, '|'),
            ''
        ) @> ARRAY[<<$SelectedErrors>>]::text[]
    )
```

这段 SQL 的意思是：

- 选择 `指定なし`：不限制错误。
- 选择 AA、BB：只保留同时包含 AA、BB 的记录。

**这一段只能粘贴到 QuickSight 的自定义 SQL 中。**

其中：

```text
<<$SelectedErrors>>
```

是 QuickSight 参数占位符，普通数据库客户端不认识它。QuickSight 会把用户选择的值传入这里；预览时则使用数据集参数的默认值。[Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-insert-parameter.html?utm_source=chatgpt.com)

另外，不要写成：

```sql
ARRAY['<<$SelectedErrors>>']
```

占位符外面不加引号。

**保存前检查：**

1. 默认值为 `指定なし` 时，预览能显示原有明细。
2. 没有“参数不存在”或 SQL 语法错误。
3. 保存并发布该数据集。

---

**第 8 步【QS】：建立测试分析，并加入两个数据集**

先新建一份测试分析，使用：

```text
Warranty_Error_AND_DQ
```

然后通过分析中的数据集管理入口——通常是数据集名称旁的菜单或 **Manage datasets**——再加入：

```text
Warranty_Error_Options
```

此时分析包含：

| 数据集 | 在分析中的用途 |
|---|---|
| `Warranty_Error_AND_DQ` | 明细表和后续统计图 |
| `Warranty_Error_Options` | 下拉框选项 |

**这是在同一个分析中添加两个数据集，不是把两份数据做 JOIN。**

接着建立一个 Table visual，使用 **`Warranty_Error_AND_DQ`**，先放入：

```text
实际记录ID
work_equip_fail_dp_w_01
work_equip_fail_dp_w_02
work_equip_fail_dp_w_03
work_equip_fail_dp_w_04
error_all
```

记录 ID 使用你真实的标识字段，并按维度显示，方便逐条核对。

先把明细表验证正确，再接回正式统计画面。

---

**第 9 步【QS】：映射分析参数**

在**分析页面**打开 **Parameters／パラメータ**。

找到数据集参数区域中的：

```text
SelectedErrors
```

然后：

1. 点击右侧 `…`。
2. 选择 **Map parameter**。
3. 选择 **Create new**。
4. 新分析参数命名为：

```text
pSelectedErrors
```

5. 静态默认值填写：

```text
指定なし
```

6. 确认映射关系指向：

```text
Warranty_Error_AND_DQ → SelectedErrors
```

7. 保存。

这里的关系是：

| 名称 | 负责什么 |
|---|---|
| `pSelectedErrors` | 接收页面上的选择 |
| `SelectedErrors` | 把选择传入 SQL |

**“映射”就是把它们连接起来，并不代表要建立两个控件。**

分析参数的默认值会覆盖数据集参数默认值，所以两边都设置为 `指定なし`。[Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-analysis.html?utm_source=chatgpt.com)

---

**第 10 步【QS】：添加唯一的多选下拉控件**

找到分析参数：

```text
pSelectedErrors
```

点击右侧 `…` → **Add control**。

设置如下：

| 配置项 | 设置 |
|---|---|
| 控件名称 | `エラー内容（選択したすべてを含む）` |
| Style | Dropdown，多选 |
| Values | Link to a dataset field |
| Dataset | `Warranty_Error_Options` |
| Field | `error_code` |
| Hide Select All | 勾选 |

如果添加控件窗口没有显示全部配置，先添加控件，再从控件右上角 `…` → **Edit** 中设置。

**候选字段必须选择 `Warranty_Error_Options.error_code`，不要选择明细里的 `error_all`。**

数据集参数映射后的多值控件不支持原生 Select all，所以这里隐藏它，使用候选列表里的 `指定なし` 表示不限制。[Amazon Quick](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-advanced-options.html?utm_source=chatgpt.com)

此时页面上应只有一个错误多选下拉控件。

**不需要再到 Filters 面板给四个错误字段创建筛选器。**筛选已经由 SQL 完成。

接下来实际操作：

1. 初始选择为 `指定なし`，明细不限制错误。
2. 先勾选两个真实错误，例如 AA、BB。
3. 再取消 `指定なし`。
4. 检查明细是否只剩同时含有 AA、BB 的记录。
5. 恢复不限制时，先勾选 `指定なし`，再取消其他错误。

这样操作也避免在切换过程中出现空选择。

> 当前 SQL 的规则是：只要勾着 `指定なし`，就不限制错误。不要把“清空所有选项”当作恢复全部数据的方法；发布前也要检查控件被清空时的实际行为。

---

**第 11 步【项目接入】：处理原有筛选器和分母**

上面的新测试分析跑通后，再接入保证延长画面。

**先处理原来的错误 OR group。**

在正式分析的副本中，用新数据集和新控件替换原来的错误筛选功能。原来的错误筛选条件如果仍然启用，就会继续叠加影响结果。

但原 group 中的：

```text
data_type_utilization_system_union 的 NULL 条件
remake_stack_leakage_judge
```

需要分别确认用途，不能按普通错误条件直接删除。

**再确认分母放在哪里。**

| 当前数据结构 | 接入方式 |
|---|---|
| 分母使用独立数据集 | 不给分母数据集映射错误参数 |
| 分子、分母通过 UNION 放在同一数据集 | 只在分子查询分支加入 AND 条件，然后与分母联合 |
| 每条错误明细上 JOIN 了分母数值 | 需要检查过滤明细后，分母聚合是否改变，不能假定它保持不变 |

如果是第二种，处理顺序应是：

1. 查询不良明细。
2. 用 `SelectedErrors` 筛选不良明细。
3. 与完整分母数据 `UNION ALL`。
4. 在 QS 中计算不良率。

**不要把第 7 步的 WHERE 直接放到包含分母的 UNION 结果最外层。**否则没有错误内容的分母行可能被删除。

你目前没有提供现有 UNION SQL 和数据类型含义，所以这一部分还不能给出可直接替换的完整查询。前面步骤可以独立跑通；接入正式不良率画面时，需要据现有 SQL 确定筛选位置。

另外，本方案目前只覆盖四个错误字段。如果第五个字段也要参与客户的“全部包含”判断，需要先确定它代表的错误值，再纳入拼接和候选列表。

---

**第 12 步【验收】：逐项检查**

用真实错误值替换 AA、BB 后，执行以下检查：

| 检查项 | 预期结果 |
|---|---|
| 只选择 `指定なし` | 错误条件不限制明细 |
| 只选择 AA | 每条返回记录至少有一个 AA |
| 选择 AA、BB | 每条返回记录同时有 AA 和 BB |
| 记录为 `BB|CC|AA` | 选择 AA、BB 时能显示 |
| 记录为 `AA|AA|CC` | 选择 AA、BB 时不能显示 |
| 四个错误全部为空 | 选择具体错误时不能显示 |
| 选择五个不同错误 | 若每条最多四种错误，应无匹配 |
| 查询后重新打开下拉框 | 完整候选列表仍然存在 |
| 与 DB 第 3 步结果对比 | 相同条件下，匹配记录一致 |
| 切换错误选择 | 分母网络连接台数保持不变 |
| 新增错误并刷新候选数据集 | 新错误进入候选列表 |

核对 DB 和 QS 结果时，先保证测试分析没有额外日期、机种等筛选条件，否则两边结果可能不同。

SQL 只负责筛选，不负责重定义“不良件数”。正式画面的分子计数方式仍沿用客户已经确定的二元表口径。

---

**遇到问题时，按这个表定位**

| 现象 | 优先检查 |
|---|---|
| DB 创建视图失败 | 来源名称、字段名、schema 创建权限 |
| DB 能查，QS 无法读取新视图 | QS 数据源使用的数据库账号是否有 schema 使用权限和视图 SELECT 权限 |
| 提示参数不存在 | 是否在明细数据集里先创建了 `SelectedErrors`，拼写是否一致 |
| DB 客户端提示 `<<` 附近语法错误 | 把 QS 参数 SQL 放进了 DB；DB 测试应使用具体错误值 |
| 只能单选 | 参数是否误建成 Single value |
| 选择 AA、BB 后仍然显示全部 | 是否仍勾着 `指定なし` |
| 控件变化，明细没有变化 | 是否完成参数映射；visual 是否还使用旧数据集 |
| 下拉框显示 `AA|BB|CC` | 绑定了 `error_all`，应改为候选数据集的 `error_code` |
| 下拉框只剩已选择的错误 | 候选值可能绑定到了被筛选的明细数据集 |
| 新错误没出现在下拉框 | 刷新 `Warranty_Error_Options` 的 SPICE 数据 |
| 不良率的分母变小 | 错误条件影响到了分母记录或分母聚合 |

**建议你实际操作时先完成第 1～3 步，确认 DB 能准确查出同时含有两个错误的记录；随后按第 4～10 步把同一判断连接到一个 QS 多选下拉框。**
