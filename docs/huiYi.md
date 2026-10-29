F
可以，我们就围绕一个例子说明：

> **用户在一个下拉框中勾选 AA 和 BB → 页面只显示同时含有 AA、BB 的记录。**

你的“把四个错误字段合并成一个字段”的想法可以用。整个方案只有三部分：**合并字段、下拉框、负责判断的 SQL。**

**一、先看数据库里准备什么**

假设原来的数据是：

| 记录ID | 错误① | 错误② | 错误③ | 错误④ |
|---|---|---|---|---|
| 001 | AA | BB | CC | 空 |
| 002 | BB | CC | AA | 空 |
| 003 | AA | CC | 空 | 空 |
| 004 | BB | DD | 空 | 空 |

按你的想法，新增一个字段 `error_all`，把非空错误用 `|` 连接：

| 记录ID | error_all |
|---|---|
| 001 | `AA|BB|CC` |
| 002 | `BB|CC|AA` |
| 003 | `AA|CC` |
| 004 | `BB|DD` |

**这一部分只是整理数据，还没有做筛选。**可以在现有 SQL 或视图中生成这个字段，不一定要修改原表。

---

**二、下拉框负责收集“用户想找什么”**

页面上仍然只有一个多选下拉框，例如：

```text
エラー内容
☐ 指定なし
☑ AA
☑ BB
☐ CC
☐ DD
```

用户勾选 AA 和 BB 后，QuickSight 就记住：

```text
用户选择的错误 = AA、BB
```

这里的“保存用户选择的变量”，就叫**参数**。

你可以把参数理解为一个小盒子：

> 下拉框选中了什么，盒子里就装什么。

我们给这个盒子取名：

```text
SelectedErrors
```

**它不需要为每种错误分别建立参数。几百种候选错误，也只用这一个多值参数。**

下拉框里的候选项，应当是独立的：

```text
AA
BB
CC
DD
……
```

而不是拼接后的：

```text
AA|BB|CC
BB|CC|AA
……
```

因此，需要一份只有一列的“错误候选列表”。它只负责提供选项，不参与明细计数。

---

**三、SQL 负责判断“这条记录是否包含所有选择”**

用户选中 AA、BB 后，对于每一条记录，数据库都检查：

> 这条记录有没有 AA？  
> 有没有 BB？  
> **两者都有，才保留。**

结果就是：

| 记录ID | error_all | 有没有 AA | 有没有 BB | 最终结果 |
|---|---|---|---|---|
| 001 | `AA|BB|CC` | 有 | 有 | ✅ 保留 |
| 002 | `BB|CC|AA` | 有 | 有 | ✅ 保留 |
| 003 | `AA|CC` | 有 | 没有 | ❌ 排除 |
| 004 | `BB|DD` | 没有 | 有 | ❌ 排除 |

**因此，AA、BB 存放顺序不同也没有关系，中间隔着其他错误也没有关系。**

实现这个判断，核心只有下面这一句：

```sql
string_to_array(error_all, '|')
    @> ARRAY['AA', 'BB']::text[]
```

拆开理解：

| SQL 部分 | 意思 |
|---|---|
| `string_to_array(error_all, '|')` | 按 `|` 把字符串拆成一个个错误 |
| `ARRAY['AA', 'BB']::text[]` | 要查找的错误列表：AA、BB |
| `@>` | 左边是否包含右边的全部内容 |

例如记录 002：

```text
原始字符串：
BB|CC|AA

拆开之后：
BB、CC、AA

要查找：
AA、BB

是否全部包含：
是 → 保留
```

这里的 `@>` 是 PostgreSQL 的“全部包含”运算符。[docs.aws.amazon.com](https://www.postgresql.org/docs/current/functions-array.html?utm_source=chatgpt.com)

---

**四、如何让 SQL 使用下拉框里的选择？**

刚才为了说明，我们在 SQL 里写死了：

```sql
ARRAY['AA', 'BB']::text[]
```

实际使用时，客户可能选择其他错误，所以改成：

```sql
ARRAY[<<$SelectedErrors>>]::text[]
```

其中：

```text
<<$SelectedErrors>>
```

意思就是：

> **把下拉框选择的错误放到这里。**

可以理解为 QuickSight 自动完成下面的替换：

| 下拉框勾选内容 | SQL 中对应的错误列表 |
|---|---|
| AA | `ARRAY['AA']::text[]` |
| AA、BB | `ARRAY['AA','BB']::text[]` |
| BB、CC、DD | `ARRAY['BB','CC','DD']::text[]` |

**你只写一次 SQL。以后客户选择什么，QuickSight 就传入什么，不需要你修改 SQL。**AWS 支持通过这种占位符把多值数据集参数传入自定义 SQL。[docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-insert-parameter.html?utm_source=chatgpt.com)

完整模板如下：

```sql
SELECT t.*
FROM your_view AS t
WHERE
    '指定なし' IN (<<$SelectedErrors>>)
    OR
    array_remove(
        string_to_array(t.error_all, '|'),
        ''
    ) @> ARRAY[<<$SelectedErrors>>]::text[]
```

这里的 `your_view` 替换为实际表名或视图名。

这段 SQL 用中文表达就是：

```text
如果用户选择了“指定なし”：
    不限制错误内容。

否则：
    把每条记录的 error_all 拆开，
    只保留包含所有选中错误的记录。
```

`array_remove(..., '')` 只是清除拆分后的空内容，不影响 AND 的核心判断。

---

**五、为什么需要 Direct Query？**

因为 SQL 需要知道客户**此刻选中了什么**。

操作过程是：

1. 用户在下拉框勾选 AA、BB。
2. QuickSight 把 AA、BB 传给 SQL。
3. PostgreSQL 执行 SQL，找出符合条件的记录。
4. QuickSight 用这些记录更新图表。

这就是 **Direct Query：根据当前选择查询数据库**。

如果使用 SPICE，数据已经提前导入 QuickSight，点击控件不会重新执行这段数据源 SQL。因此，这个方案的明细数据集需要使用 Direct Query。[docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters.html?utm_source=chatgpt.com)

---

**六、实际配置时，你要做哪些操作？**

按下面的顺序即可：

| 顺序 | 要做的事 | 目的 |
|---|---|---|
| ① | 在数据中增加 `error_all` | 把四个错误字段集中到一个字段 |
| ② | 准备完整的错误候选列表 | 给下拉框提供 AA、BB 等选项 |
| ③ | 明细数据集选择 Direct Query | 允许根据用户选择执行 SQL |
| ④ | 创建多值数据集参数 `SelectedErrors` | 让 SQL 可以接收错误列表 |
| ⑤ | 在自定义 SQL 中加入上面的判断 | 实现“全部包含” |
| ⑥ | 在分析中映射这个参数，并添加一个多选下拉控件 | 让用户的选择传到 SQL |

第⑥步提到的“映射”只是**把页面里的选择和 SQL 参数连接起来**。虽然配置时会看到“分析参数”和“数据集参数”两个名称，**页面上仍然只有一个下拉框**。[docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/dataset-parameters-analysis.html?utm_source=chatgpt.com)

另外保留两个规则：

- 选择具体错误时，取消 `指定なし`；恢复不限制时，只勾选 `指定なし`。
- 这段错误筛选应作用于不良明细，不能把网络连接台数的分母记录一起过滤掉。

**你现在可以先只验证最核心的一步：拿合并后的 `error_all`，在数据库里运行下面这个固定条件。**

```sql
SELECT *
FROM your_view
WHERE string_to_array(error_all, '|')
      @> ARRAY['AA', 'BB']::text[]
```

将 AA、BB 替换为两个真实错误值。只要返回的每条记录都同时包含这两个错误，就说明判断逻辑已经成立；后面的 QuickSight 配置，只是把这里写死的两个值换成用户在下拉框中的选择。
