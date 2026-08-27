对，你记得没错。最近会议里提到的「コントロール」，大概率是指 QuickSight 页面上的筛选控件（Filter Control）。

会议要求可以理解为：

* 在正式环境的 Analysis 中先创建页面框架。
* 配置 Dataset 和 Sheet。
* 参考现有 QuickSight 页面，设置相同或类似的 Filter / Control。
* Control 就是用户可以在页面上操作的下拉框、日期选择框等。
* 暂时只做 Analysis，不需要发布成 Dashboard。
* 页面上先放置要求的三个图表。

另外，Control 对数据的影响需要注意：

* 机种、月份等 Control：会影响显示范围，部分情况下也会影响分母。
* Error Code、Error Type：主要筛选分子，不应该导致分母随错误项目一起缩小。
* 如果两个图表使用不同 Dataset，即使字段名称相同，一个普通 Filter Control 通常不能直接同时控制两个 Dataset；可能需要分别建立过滤器，或者通过 Parameter 实现联动。
* 这也是会议倾向于把三张表先整合到同一个 View，再作为 QuickSight Dataset 使用的原因之一。

所以，你制作 QS 页面时，不能只放图表，还需要把页面顶部或合适位置的筛选下拉框等 Control 一起做出来，并尽量参考对方给你的现有 QS 页面布局。
