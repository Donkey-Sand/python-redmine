可以使用 QuickSight 的跨数据集过滤器（Cross-dataset filter）。不需要把两个 Dataset JOIN 到一起。

## 设置步骤

假设两个 Dataset 都有字段：

```text
target_month
series_name
gas_type
err_code
err_type
```

以 `series_name` 为例：

1. 打开 QuickSight Analysis。
2. 进入对应的 Sheet。
3. 左侧打开「Filters／フィルター」。
4. 点击「Add filter／フィルターを追加」。
5. 从 Dataset A 选择：

```text
series_name
```

6. 编辑该 Filter，在「Applied to／適用先」中选择：

```text
Single sheet／このシート
```

或者：

```text
All applicable visuals／適用可能なすべてのビジュアル
```

7. 勾选：

```text
Apply cross-datasets
データセット間に適用
```

8. 在 Filter 的菜单中选择：

```text
Add control → Inside this sheet
コントロールを追加 → このシート内
```

这样生成的一个 `series_name` 筛选控件，就可以同时控制：

```text
Visual A → Dataset A
Visual B → Dataset B
```

AWS 官方说明中，跨数据集过滤器可以将同一个 Filter 应用到不同 Dataset 的所有适用 Visual。[QuickSight 跨数据集过滤说明](https://docs.aws.amazon.com/quick/latest/userguide/cross-sheet-filters.html)

## 字段必须满足的条件

两个 Dataset 中的对应字段必须：

* 字段名称完全一致；
* 大小写完全一致；
* 空格和符号完全一致；
* QuickSight 数据类型一致。

例如：

| Dataset A            | Dataset B             | 是否可以自动映射 |
| -------------------- | --------------------- | -------- |
| `series_name` String | `series_name` String  | 可以       |
| `series_name` String | `Series_Name` String  | 不可以      |
| `target_month` Date  | `target_month` String | 不可以      |
| `err_code` String    | `err_code` Integer    | 不可以      |

QuickSight 会根据“完全相同的字段名和数据类型”自动映射两个 Dataset 的字段。[AWS 字段映射说明](https://docs.aws.amazon.com/quick/latest/userguide/mapping-and-joining-fields.html)

如果名称不同，可以在其中一个 Dataset 创建计算字段，例如：

```text
series_name
```

内容为：

```text
{model_series}
```

让最终字段名称和数据类型与另一个 Dataset 保持一致。

## 你的 Sheet 推荐这样设置

分别创建下面几个 Filter，并全部勾选 `Apply cross-datasets`：

```text
target_month
series_name
gas_type
err_code
err_type
```

最终结构为：

```text
target_month Filter ─┬─ Dataset A 的 Visual
                     └─ Dataset B 的 Visual

series_name Filter ──┬─ Dataset A 的 Visual
                     └─ Dataset B 的 Visual
```

也就是说，一个筛选控件可以控制两个 Dataset，但每个筛选字段仍需分别创建一个 Filter。

如果勾选 `Apply cross-datasets` 后，第二个 Visual 没有变化，最常见原因就是：

1. 字段名不完全相同；
2. 字段数据类型不同；
3. Filter 的 Applied to 只选择了第一个 Visual；
4. 第二个 Visual 使用的是计算字段，而不是被自动映射的原始字段。
