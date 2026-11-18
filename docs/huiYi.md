你可以直接在 **QuickSight 的分析画面**里完成。现有的「交換部品」多选下拉框继续使用，几百个部品名都由参数 `replacePartsBinaryParameter` 自动传入。你只需要做两件事：**新增一个计算字段，再把它放进现有的部品 OR 筛选组。** [docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/in-function.html?utm_source=chatgpt.com)

## 第 1 步：先确认 32 列的编号

在你截图左侧的 **「データ」** 面板里，选中当前这个 `v_sum_fhsbu_warranty_extension_defect_rate_monthly_...` 的 SPICE 数据集，在字段搜索框里输入 `replace_parts_nm_`，往下滚到底。

截图里可以确认字段按 `01_w_01`、`01_w_02`、`01_w_03`、`01_w_04` 这样的规则排列。**下面的完整公式按 `01～08`，每组 `w_01～w_04`，共 32 列编写。** 请确认最后一个字段是：

```text
replace_parts_nm_08_w_04_utilization_system_union
```

如果末尾编号不同，以你画面里的实际字段名为准。另一个容易弄错的地方是，**你之前添加筛选条件的顺序和数据面板的排序不同**。所以这次让公式覆盖全部 32 列，这样不用逐个核对原来 20 条里到底配了哪些。

## 第 2 步：新增计算字段，粘贴完整公式

在当前**分析编辑画面**选择 **「追加」→「計算フィールドを追加」**。如果你的界面把入口放在左侧，则在「データ」面板里找「計算フィールドを追加」。

**字段名**填：

```text
replace_parts_any_match
```

**公式**粘贴下面这一整段。`in()` 会检查当前行的这一列是否等于用户勾选的任意一个部品名。32 个 `in()` 用 `OR` 相连，所以任何一列命中，整行就返回 `一致`。[docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/in-function.html?utm_source=chatgpt.com)

```text
ifelse(
    in({replace_parts_nm_01_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_01_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_01_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_01_w_04_utilization_system_union}, ${replacePartsBinaryParameter})

    OR in({replace_parts_nm_02_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_02_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_02_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_02_w_04_utilization_system_union}, ${replacePartsBinaryParameter})

    OR in({replace_parts_nm_03_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_03_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_03_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_03_w_04_utilization_system_union}, ${replacePartsBinaryParameter})

    OR in({replace_parts_nm_04_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_04_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_04_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_04_w_04_utilization_system_union}, ${replacePartsBinaryParameter})

    OR in({replace_parts_nm_05_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_05_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_05_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_05_w_04_utilization_system_union}, ${replacePartsBinaryParameter})

    OR in({replace_parts_nm_06_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_06_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_06_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_06_w_04_utilization_system_union}, ${replacePartsBinaryParameter})

    OR in({replace_parts_nm_07_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_07_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_07_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_07_w_04_utilization_system_union}, ${replacePartsBinaryParameter})

    OR in({replace_parts_nm_08_w_01_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_08_w_02_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_08_w_03_utilization_system_union}, ${replacePartsBinaryParameter})
    OR in({replace_parts_nm_08_w_04_utilization_system_union}, ${replacePartsBinaryParameter}),
    '一致',
    '不一致'
)
```

点 **「保存」**。如果某个字段名标红，删掉标红的 `{字段名}`，从编辑器的字段列表中点击对应字段重新插入。如果标红的是参数，从编辑器的参数列表中插入现有的 `replacePartsBinaryParameter`。

## 第 3 步：在原来的部品组里腾出一个位置

1. 打开 **「フィルター」** 面板，找到你截图下半部分的 **交換部品组**。认准里面的条件都写着 `replace_parts_nm_...` 和 `replacePartsBinaryParameter`。
2. 展开这个组，找到任意一个部品条件，例如 `replace_parts_nm_05_w_03_utilization_system_union`。
3. 点击**该字段这一行右侧**的垃圾桶，移除这一条。此时组里还剩 19 条。新公式已经覆盖这列，所以可以移除。
4. **保留上方的 `errorcontent` 组，也不要点底部的「すべて削除」。**

## 第 4 步：把新字段加回同一个 OR 组

这是操作里最重要的一步。**点击部品组内部、其他条件下面的「フィルター条件を追加」**。这样新条件和已有部品条件之间显示的才是 **「または」**。如果从面板顶层新建独立筛选，它和旧组之间会变成 **「および」**，只在新增列中命中的记录就可能被旧组挡掉。[docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/add-a-compound-filter.html?utm_source=chatgpt.com)

选中字段 `replace_parts_any_match`，按下表设置：

| 画面项目 | 设置值 |
|---|---|
| フィルタータイプ | `カスタムフィルター` |
| フィルター条件 | `次と等しい` |
| パラメータを使用 | **不勾选** |
| 比较值 | `一致`，直接输入这两个字，不加引号 |
| 最后 | 点击 `適用` |

这里有两个不同的值：**用户在下拉框里选「スタック」等部品名，筛选器在后台判断计算结果是否等于「一致」**。因此用户看到的下拉框、候选部品名和操作方式都和现在一样。[docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/in-function.html?utm_source=chatgpt.com)

## 第 5 步：用明细表验证

先固定机种、月份和错误内容等其他筛选，只改「交換部品」。按下面顺序测：

| 测试 | 你的操作 | 预期结果 |
|---|---|---|
| **① 最重要：验证原来漏掉的列** | 找一条已知明细，它的某个部品只出现在原先没配进筛选组的列里。下拉框只选这个部品 | 这条明细仍然显示。如果消失，先检查第 4 步新条件和旧条件之间是否为「または」 |
| **② 验证多选** | 同时勾选「スタック」和「BBファン」 | 只含「スタック」的行、只含「BBファン」的行，以及两者都有的行，都能显示 |
| **③ 验证精确匹配** | 只勾「BBファン」 | 如果某行只有「BBファンカンセイ」，其余列也没有「BBファン」，该行应被排除。`in()` 按完整值匹配 [docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/in-function.html?utm_source=chatgpt.com) |
| **④ 验证全选** | 点击「すべて選択」，与相同机种、月份和错误条件下未限制部品时的件数对照 | 如果业务定义是“全选显示全部明细”，两边件数应一致。若有 32 列全部为空的明细，也单独核对一条。QuickSight 对「すべて選択」使用参数保留值，需要单独验证 [docs.aws.amazon.com](https://docs.aws.amazon.com/quick/latest/userguide/parameters-set-up.html?utm_source=chatgpt.com) |

**完成后的画面特征**：上方错误内容组和下方部品组之间显示 **「および」**；下方部品组内，新字段与旧条件之间显示 **「または」**；新条件显示 **`replace_parts_any_match` →「次と等しい - 一致」**。这样就保留了你原先的组合逻辑，同时覆盖全部 32 个部品字段。
