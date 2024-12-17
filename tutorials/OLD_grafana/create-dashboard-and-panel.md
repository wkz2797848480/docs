---
标题: 创建格拉法纳（Grafana）仪表盘和面板
摘要: 在格拉法纳（Grafana）中对数据进行可视化展示。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析]
---

# 创建Grafana仪表板和面板

Grafana通过“仪表板”和“面板”进行组织。仪表板代表了对系统性能的视图，每个仪表板由一个或多个面板组成，这些面板提供了与该系统相关的特定指标的信息。

在本教程中，您将构建一个简单的仪表板，将其连接到TimescaleDB，并可视化数据。

## 前提条件

在开始之前，请确保您已经：

*   [安装TimescaleDB][install-timescale]。
*   设置Grafana。

当您的TimescaleDB和Grafana安装完成后，摄取[纽约出租车][nyc-taxi]教程中的数据，并将Grafana配置为连接到该数据库。

## 构建新仪表板

首先创建一个新的仪表板。在Grafana用户界面的最左侧，您会看到一个`+`图标。当您悬停时，会看到一个`Create`菜单，其中有一个`Dashboard`选项。选择该`Dashboard`选项。

创建新仪表板后，您将看到一个`New Panel`屏幕，提供`Add Query`和`Choose Visualization`选项。将来，如果您已经有了带有面板的仪表板，您可以点击Grafana用户界面顶部的`+`图标，这将允许您向现有仪表板添加面板。

要继续本教程，请通过点击`Choose Visualization`选项添加新的可视化。

此时，您有几个不同的Grafana可视化选项。选择第一个选项，`Graph`可视化。

![Grafana可视化选项](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_visualizations.png)

有多种方法可以配置面板，但您可以接受所有默认设置并创建一个简单的`Lines`图表。

在Grafana用户界面的最左侧部分，选择`Queries`标签。

![如何创建新的Grafana查询](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/create_grafana_query.png)

而不是使用Grafana查询构建器，直接编辑查询。在视图中，点击底部的`Edit SQL`按钮。

![在Grafana中编辑自定义SQL查询](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/edit_sql_in_grafana.png)

在开始编写查询之前，您还需要将查询数据库设置为您之前连接的纽约市出租车数据集：

![在Grafana中切换数据源](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/set_data_source.png)

<Highlight type="note">
如果您在Grafana中可视化时间序列数据，请确保在查询构建器的`Format As`下拉菜单中选择`Time series`。
</Highlight>

### 在TimescaleDB中可视化存储的指标

首先，创建一个可视化，回答[纽约出租车][nyc-taxi]教程中的“每天发生了多少次行程？”这个问题。

从教程中，您可以看到我们的查询的标准SQL语法：

```sql
SELECT date_trunc('day', pickup_datetime) AS day,
  COUNT(*)
FROM rides
GROUP BY day
ORDER BY day;
```

您需要修改这个查询以支持Grafana独特的查询语法。

#### 修改SELECT语句

首先，修改`date_trunc`函数以使用TimescaleDB的`time_bucket`函数。您可以查阅TimescaleDB的[API参考time_bucket][time-bucket-reference]以获取如何正确使用它的更多信息。

查看这个查询的`SELECT`部分。首先，使用`time_bucket`函数将结果分组到一天的分组中。如果您将Grafana面板的`Format`设置为`Time series`，例如用于图表面板，那么查询必须返回一个名为`time`的列，该列返回SQL `datetime`或代表Unix纪元的任何数值数据类型。

因此，这部分新查询的第一部分被修改为将`time_bucket`分组的输出标记为`time`，因为Grafana需要，而第二部分保持不变：

```sql
SELECT
  --1--
  time_bucket('1 day', pickup_datetime) AS "time",
  --2--
  COUNT(*)
FROM rides
```

#### Grafana timeFilter函数

Grafana时间序列面板包括一个工具，允许您在给定的时间范围内进行过滤，称为时间过滤器。不足为奇的是，Grafana有一种方法可以将Grafana面板中的用户界面构造与查询本身链接起来；在这种情况下，是`$__timefilter()`函数。

在这个修改后的查询示例中，使用`$__timefilter()`函数将`pickup_datetime`列设置为您的可视化的过滤范围：

```sql
SELECT
  --1--
  time_bucket('1 day', pickup_datetime) AS "time",
  --2--
  COUNT(*)
FROM rides
WHERE $__timeFilter(pickup_datetime)
```

#### 引用查询中的元素

最后，您可以根据您选择的时间桶对可视化结果进行分组，也可以根据时间桶对结果进行排序。因此，`GROUP BY`和`ORDER BY`语句引用`time`。

有了这些更改，这是最终的Grafana查询：

```sql
SELECT
  --1--
  time_bucket('1 day', pickup_datetime) AS time,
  --2--
  COUNT(*)
FROM rides
WHERE $__timeFilter(pickup_datetime)
GROUP BY time
ORDER BY time
```

当您在Grafana中可视化这个查询时，您将看到：

![在Grafana中可视化时间序列数据](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_query_results.png)

<Highlight type="note">
记住在Grafana仪表板的右上角设置时间过滤器。如果您使用的是本例的预构建样本数据集，您可以将时间过滤器设置在2016年1月1日左右。
</Highlight>

目前，数据被分组到1天的分组中。调整`time_bucket`函数，改为5分钟的分组，并比较图表：

```sql
SELECT
  --1--
  time_bucket('5m', pickup_datetime) AS time,
  --2--
  COUNT(*)
FROM rides
WHERE $__timeFilter(pickup_datetime)
GROUP BY time
ORDER BY time
```

当您可视化这个查询时，它看起来像这样：

![在Grafana中可视化时间序列数据](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_query_results_5m.png)

### 总结

通过遵循所有TimescaleDB + Grafana教程，完成您的Grafana知识。

[install-timescale]: /getting-started/latest/
[nyc-taxi]: /tutorials/:currentVersion:/nyc-taxi-cab
[time-bucket-reference]: /api/:currentVersion:/hyperfunctions/time_bucket

