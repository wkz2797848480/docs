---
标题: 使用 Tableau 可视化 TimescaleDB 中的数据
摘要: 使用 Tableau 绘制图表并对数据进行可视化展示。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [可视化，分析，Tableau]
---

# 使用Tableau可视化TimescaleDB中的数据

[Tableau][get-tableau]是一个流行的分析平台，使您能够深入了解您的业务。它是可视化存储在[TimescaleDB][timescale-products]中的数据的理想工具。

本教程涵盖：

*   设置Tableau以与TimescaleDB配合使用
*   从Tableau运行TimescaleDB查询
*   在Tableau中可视化数据

### 前提条件

要完成本教程，您需要对结构化查询语言（SQL）有基本了解。虽然教程会逐步引导您了解每个SQL命令，但如果您之前接触过SQL，将会有所帮助。

首先，[安装TimescaleDB][install-timescale]。安装完成后，您可以继续导入或创建样本数据并完成教程。

另外，[获取Tableau的副本或许可证][get-tableau]。

您还希望[完成加密货币教程][crypto-tutorial]，因为它设置并配置了您需要完成本教程其余部分的数据。您可以可视化在加密货币教程末尾找到的许多查询。

### 第1步：设置Tableau以连接到TimescaleDB

找到您的TimescaleDB实例的`host`、`port`和`password`。

将您的TimescaleDB实例连接到Tableau只需几次点击，这要归功于Tableau内置的Postgres连接器。要连接到您的数据库，请添加一个新连接，在“连接到服务器”部分选择PostgreSQL作为连接类型。然后输入您的数据库凭据。

### 第2步：在Tableau中运行简单查询

让我们使用Tableau中的内置SQL编辑器。要运行查询，通过拖放“新自定义SQL”按钮（在Tableau桌面用户界面的左下角）到“在此处拖动表”位置，添加自定义SQL到数据源。

在对话框中输入查询。在这种情况下，使用[加密货币教程][crypto-tutorial]中的第一个查询：

```sql
SELECT time_bucket('7 days', time) AS period,
      last(closing_price, time) AS last_closing_price
FROM btc_prices
WHERE currency_code = 'USD'
GROUP BY period
ORDER BY period
```

您应该在Tableau中看到与在`psql`命令行中运行查询时相同的结果。

我们还将数据源命名为`btc_7_days`，如下所示。

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-tableau-tutorial/tableau-simple-query-results.png" alt="使用Tableau查看时间序列数据"/>

### 第3步：在Tableau中可视化数据

表格中的结果仅有一定的用处，图形效果更佳！因此，在最后一步中，让我们将前一步的输出转换为Tableau中的交互式图形。

为此，创建一个新的工作表（或仪表板），然后选择您想要的数据源（在我们的例子中是`btc_7_days`），如下所示。

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-tableau-tutorial/tableau-new-worksheet.png" alt="在Tableau中检查时间序列数据的新工作表"/>

在最左侧的窗格中，您将看到Tableau称之为“维度”和“度量”的部分。每当您使用Tableau时，它会将您的字段分类为维度或度量。度量是一个依赖变量，意味着其值是一个或多个维度的函数。例如，某一天某个项目的价格是基于那一天的度量。因此，给定的日期不会根据我们数据库中的任何其他值而改变。

更直接地说，1776年7月4日仍然是1776年7月4日，即使茶的价格飙升。然而，茶的价格可能会变化，具体取决于你查看的日期。

因此，在这种情况下，将维度`period`移动到工作表的列部分，并检查`last_closing_price`度量，取决于给定的`period`。在Tableau中，您可以将这些元素拖放到适当的位置，如下所示：

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-tableau-tutorial/tableau-dimension-measure-setup.png" alt="在Tableau中检查时间序列数据的新维度和度量"/>

现在，这个图形的精度不足，因为数据点按年分组。要修复此问题，单击`period`上的下拉箭头并选择“确切日期”。

<img class="main-content__illustration" src="https://assets.iobeam.com/images/docs/screenshots-for-tableau-tutorial/tableau-granular.png" alt="在Tableau中分析详细数据以检查时间序列数据"/>

Tableau是一个强大的商业智能工具，是存储在TimescaleDB中的数据的理想伴侣。本教程仅触及了使用Tableau可视化数据的表面。

[crypto-tutorial]: /tutorials/:currentVersion:/blockchain-analyze/
[get-tableau]: https://www.tableau.com/products/trial 
[install-timescale]: /getting-started/latest/
[timescale-products]: https://www.timescale.com/products/

