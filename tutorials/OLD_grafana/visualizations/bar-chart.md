---
标题: 在格拉法纳（Grafana）中构建柱状图
摘要: 在格拉法纳（Grafana）中绘制柱状图，以对比不同类别间的值。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析]
标签: [柱状图]
---

import GrafanaVizPrereqs from 'versionContent/_partials/_grafana-viz-prereqs.mdx';

# 在Grafana中构建条形图

条形图使用条形显示数据，每个条形代表一个特定类别。条形图是数据可视化中比较数据的绝佳工具，因为可以很容易看出哪个项目的柱状图或条形更长或更短。条形图使用两个轴：垂直和水平。条形越长，其值越大。条形图是比较多组之间项目的好方法。

条形图可以回答以下问题：

*   今天哪只股票交易量最高？
*   上周股票的交易量分布如何？
*   A年级中有多少学生在某个特定年龄范围内？

要绘制条形图，Grafana只需要一个数据帧。您需要至少有一个字符串字段，用作X轴或Y轴的类别，以及一个或多个数值字段。如果您想在单个面板中绘制多个条形图，还可以提供多个值列。

本教程向您展示如何：

*   使用`time_bucket()`创建预聚合数据的条形图。
*   在单个面板中创建多个条形图。
*   使用预聚合数据创建堆叠条形图。

可以选择几种不同类型的条形图，包括垂直、水平和堆叠条形图。本教程涵盖了所有这些类型。

## 前提条件

<GrafanaVizPrereqs />

查看这个视频，了解在Grafana中创建条形图的逐步操作：

<Video url="https://www.youtube-nocookie.com/embed/nBcUiPvYjhc&#34;/&gt; 

## 使用预聚合数据创建条形图

使用表`stocks_real_time`中的数据创建条形图可视化。

<Procedure>

### 使用预聚合数据创建条形图

1.  在查询编辑器中，使用此SQL查询条形图数据集。使用变量`$bucket interval`表示条形图覆盖的时间段：

    ```sql
    SELECT
        time_bucket('$bucket_interval', time) AS time,
        symbol,
        AVG(price) AS price
    FROM stocks_real_time srt
    WHERE symbol IN ($symbol)
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time), symbol
    ORDER BY time;
    ```

2.  在Grafana仪表板中，在`Dashboard variable`字段中选择要绘制的股票。如需要，调整仪表板的时间范围。确保返回的数据有一个名为`time`的列，包含时间戳。时间戳应按升序排列。否则，您将获得错误。返回的数据看起来像这样：

    ![Google股票有效时间序列数据的表视图截图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/barchart/Tabledataforgoogle.png)

3.  在`Visualizations`字段中，选择`Bar chart`。Grafana将查询转换为条形图。此示例显示了Google股票价格在特定时期内的垂直条形图分布，范围在$2836到$2108之间：

    ![Grafana生成的垂直条形图截图。垂直条形图代表过去2个月Google的价格](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/barchart/googlebarchart.png)

4.  您可以将垂直条形图转换为水平条形图，以便为垂直轴上的较长标签腾出空间。在仪表板中，导航到条形图部分。在`Orientation`部分，点击`horizontal`。水平条形图如下所示：

    ![Grafana生成的水平条形图截图。水平条形图代表过去2个月Google的价格](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/barchart/horizontalbarchartview.png)

</Procedure>

## 创建多个条形图

如果您想比较多个不同股票的分布，您可以创建一个包含多个条形图的面板。数据库返回所有选定值的交易，Grafana将它们分别放入不同的条形图中。

<Procedure>

### 在单个面板中创建多个条形图

1.  使用此查询从[入门教程][gsg-data]的数据集中获取所有公司符号：

    ```sql
    SELECT
        DISTINCT symbol FROM company ORDER BY symbol ASC;
    ```

2.  在Grafana仪表板中，导航到您的面板设置页面。在`Variables`部分：

    *   在`Name`字段中，给您的符号一个名称。
    *   在`Type`字段中，选择`Query`。
    *   在`Selection options`字段中，启用`multi-value`。

3.  在psql提示符下，更新之前的查询，允许您使用多个符号。您可以选择任意多的符号进行比较：

    ```sql
    SELECT
        time_bucket('$bucket_interval', time) AS time, symbol,
        AVG(price) AS price
    FROM stocks_real_time srt
    WHERE symbol IN ($symbol)
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time), symbol
    ORDER BY time;
    ```

4.  在Grafana中，刷新仪表板。返回的数据看起来像这样：

    ![四个不同股票的有效时间序列数据的表视图截图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/barchart/tableviewfivestockdata.png)

    显示的图表如下所示：

    ![Grafana生成的多个条形图截图。多个条形图代表过去1个月四种不同股票的价格](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/barchart/multiplebarchart.png)

5.  在您刚刚创建的图表中，您可以看到5种不同的股票，但很难区分它们。要区分它们，您可以通过点击每个线左侧的图例，并为每个条形选择颜色来调整每种股票的颜色：

    ![Grafana图表截图，显示5个条形图的股票值分别为绿色、蓝色、红色、紫色和橙色](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/barchart/multicoloredbarchart.png)

</Procedure>

## 创建堆叠条形图

您可以使用堆叠条形图显示不同股票如何划分为更小的类别，以及每个部分对总数的影响。

前面的示例使用了价格交易的垂直、水平和多个条形图。在本节中，您将使用桶间隔查看每只股票的交易量。

`stock_real_time`超表包含一个列，记录每日累计交易量。这有助于计算每个桶的数据量。

<Procedure>

### 创建堆叠条形图

1.  在psql提示符下，更新之前的查询，以找到符号在桶内的最大`day_volume`值。然后，从上一个桶的最大值中减去每个最大值。差值给出了该桶的交易量：

    ```sql
    SELECT
        time_bucket('$bucket_interval', time) AS time,
        symbol,
        MAX(day_volume) - LAG(MAX(day_volume), 1) OVER(
            PARTITION BY symbol
            ORDER BY time_bucket('$bucket_interval', time)
        ) AS bucket_volume
    FROM stocks_real_time srt
    WHERE symbol IN ($symbol)
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time), symbol
    ORDER BY time_bucket('$bucket_interval', time), symbol;
    ```

2.  在Grafana仪表板中，将多个条形图转换为堆叠条形图。在符号下拉菜单中，选择您想要比较的所有股票。在面板右侧，点击条形图下拉菜单。在`stacking`字段中，选择`normal`，然后刷新面板。堆叠条形图视图显示了1天的桶，桶间隔为1小时。交易量计算主要在交易日内有效：

  ![Grafana仪表板截图，显示堆叠条形图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/barchart/stackedbarcharts.png)

<Highlight type="note">
如果您超出了单个交易日，您可能会得到看起来不太好的结果，或者您可能没有数据返回。要解决这个问题，请将计算重点放在单个交易日上。
</Highlight>

</Procedure>


[gsg-data]: /getting-started/:currentVersion:/
