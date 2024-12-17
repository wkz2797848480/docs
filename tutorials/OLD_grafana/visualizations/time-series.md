---
标题: 在格拉法纳（Grafana）中构建时间序列图
摘要: 创建一个时间序列图来展示数值随时间变化的情况。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析]
标签: [时间序列]
---

import GrafanaVizPrereqs from 'versionContent/_partials/_grafana-viz-prereqs.mdx';

# 在Grafana中构建时间序列图

时间序列图是一种折线图，它绘制随时间变化的点。它允许您看到数据中的趋势和波动。通常在两个维度上绘制。x轴代表时间，y轴代表数据的值。

由于时间序列图是Grafana中最常见的图表，它也是默认的面板类型。

使用时间序列图，您可以回答以下问题：

*   今天AMD的每小时股票价格是多少？
*   过去一周每天有多少用户访问网站页面？
*   昨天的温度是多少？

## Grafana时间序列图的数据

要绘制时间序列图，Grafana需要您提供时间列和值列。要在单个面板中绘制多个时间序列图，您需要提供多个值列。

这是一个有效时间序列数据的示例：

```bash
Time                | Value_1 | Value_2 |
--------------------+---------+---------+
2022-02-08 07:30:01 |      10 |       1 |
2022-02-08 07:31:01 |      15 |       2 |
2022-02-08 07:32:01 |      20 |       3 |
2022-02-08 07:33:01 |      25 |       4 |
2022-02-08 07:34:01 |      30 |       5 |
```

本教程向您展示如何：

*   使用原始数据创建时间序列图
*   使用`time_bucket()`创建预聚合数据的时间序列图
*   在单个面板中创建多个时间序列图

## 前提条件

<GrafanaVizPrereqs />

查看这个视频，了解在Grafana中创建时间序列图的逐步操作：
<Video url="https://www.youtube-nocookie.com/embed/uRgKwcL6lDQ&#34;/&gt; 

## 使用原始数据创建时间序列图

时间序列图的一个非常常见的用例是显示股票数据。图表可以轻松看出股票价值是上升还是下降。此外，制作图表不需要额外的计算。

<Procedure>

### 使用原始数据创建时间序列图

1.  在Grafana仪表板中添加`$symbol`文本框类型的变量。

2.  在Grafana中创建一个新的面板，并添加此查询：

    ```SQL
    SELECT time,
        price
    FROM stocks_real_time
    WHERE symbol = '$symbol'
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    ORDER BY time;
    ```

3.  在`symbol`变量中输入`AMD`。如需要，调整仪表板的时间范围。此图表使用原始查询，因此如果您设置了一个较大的时间范围，Grafana可能需要很长时间来获取所有行并显示数据。

4.  当选择“Table view”时，返回的数据如下所示：

    ```bash
    time                | price |
    --------------------+--------+
    2022-02-08 06:38:21 |  123   |
    2022-02-08 06:38:28 |  123   |
    2022-02-08 06:39:18 |  123   |
    2022-02-08 06:39:56 |  123   |
                …       |   …    |
    ```

    检查您的数据是否符合Grafana绘制时间序列的要求。数据必须有一个名为`time`的列，包含时间戳。其他列可以任意命名。对于时间序列可视化，时间戳必须按升序排列。否则，您将收到错误。

5.  选择`Time series`作为您的可视化类型。

    ![Grafana仪表板截图。“Visualizations”标签被选中。在下面，“Time-series”显示为可视化类型。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/time-series-visualization-type.png)

6.  Grafana返回的图表如下所示：

    ![Grafana生成的时间序列图截图。图表代表了过去6小时AMD的价格。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/simple-time-series-graph.png)

</Procedure>

## 使用`time_bucket()`预聚合数据创建时间序列图

在之前的示例中，您查询了AMD股票在一个6小时周期内的所有交易。这大约返回了3800个数据点。如果您查询AMD股票在一个3个月周期内的所有交易，您将获得大约1,500,000个数据点。

Grafana像许多绘图工具一样，在绘制数百万点时表现不佳。此外，默认情况下，Grafana每30秒刷新一次仪表板。这进一步加大了CPU、内存和网络带宽的压力。在极端情况下，Grafana会冻结。

为了解决这个问题，您可以使用TimescaleDB的[`time_bucket`][time_bucket]超函数预聚合您的数据。

<Procedure>

## 使用`time_bucket()`预聚合数据创建时间序列图

1.  在Grafana仪表板中添加`$bucket_interval`间隔类型的变量。

2.  在Grafana中创建一个新的面板，并添加此查询：

    ```sql
    SELECT time_bucket('$bucket_interval', time) AS time,
     AVG(price) AS price
    FROM stocks_real_time
    WHERE symbol = '$symbol'
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time)
    ORDER BY time;
    ```

3.  使用`bucket_interval`为30分钟和日期范围为`Last 30 days`，Grafana返回此图表：

    ![Grafana使用`time_bucket`超函数生成的时间序列图截图。图表代表了过去30天AMD股票的价格，间隔为30分钟。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/time-bucket-graph.png)

    因为股票市场只在工作日的上午9:30到下午4:00开放，所以数据集中有大量没有数据的空白区域。Grafana自动将最后一个非空值连接到最近的另一个非空值。这在图表中创建了您看到的长、直、几乎水平的线。

4.  要解决这个问题，您可以使用Grafana的`Connect null values`设置。但首先，您需要在没有数据的地方包含空值的行。默认情况下，`time_bucket`如果不返回数据，则不返回行。在您的查询中，用[`time_bucket_gapfill`](/api/latest/hyperfunctions/gapfilling/time_bucket_gapfill/)替换`time_bucket`。如果您不指定填充函数，`time_bucket_gapfill`会在没有数据的地方返回一个空值的行。

5.  在您的查询中，用`time_bucket_gapfill`替换`time_bucket`。

    ```SQL
    SELECT time_bucket_gapfill('$bucket_interval', time) AS time,
     AVG(price) AS price
    FROM stocks_real_time
    WHERE symbol = '$symbol'
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket_gapfill('$bucket_interval', time)
    ORDER BY time;
    ```

6.  在选项面板中，将`Connect null values`设置为`Threshold`。给`Threshold`一个24小时的值。

    ![Grafana时间序列选项面板中“Connect null values”的截图。所选值为“Threshold”，并且它的值小于24小时。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/connect-null-values.png)

7.  Grafana返回的图表如下所示：

    ![Grafana使用`time_bucket_gapfill`超函数生成的时间序列图截图。图表代表了过去30天AMD股票的价格，间隔为30分钟，并且对于超过24小时的每个空白区域，空值为null。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/time-bucket-gapfill-graph.png)

    这个图表允许您更好地可视化一周内的股票价格。它填补了下午4:00到上午9:30之间不到24小时的空白，但不连接周末的值。

</Procedure>

## 在单个面板中创建多个时间序列图

如果您想比较两只股票随时间的价格，您可以制作两个单独的面板，每个面板有两个单独的符号变量。一个更好的选择是将两个时间序列图合并到一个面板中。为此，将`$symbol`变量更改为多值答案，并对您的查询进行轻微更改。

<Procedure>

## 在单个面板中创建多个时间序列图

1.  将`$symbol`变量更改为`Query`类型。

    ![“symbol”变量设置的截图。变量类型选项已选择“Query”。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/symbol-query-type.png)

2.  在查询选项中，添加以下查询。在选项中，选择`Multi-Value`。

    ```SQL
    SELECT DISTINCT(symbol) FROM company ORDER BY symbol ASC;
    ```

3.  在`Preview of values`下，您会按字母顺序看到一些公司符号。

    ![“symbol”变量设置中“Preview of values”的截图。显示了一些公司符号。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/preview-values.png)

4.  在仪表板面板中，更改查询以允许多值答案：

    ```SQL
    SELECT time_bucket_gapfill('$bucket_interval', time) AS time,
        AVG(price) AS price,
        symbol
    FROM stocks_real_time
    WHERE symbol IN ($symbol)
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket_gapfill('$bucket_interval', time), symbol
    ORDER BY time;
    ```

5.  从符号变量中选择多个股票

    ![符号变量选择器的截图。选择了'AAPL'和'AMD'两个符号。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/select-stock.png)

6.  Grafana返回的图表如下所示：

    ![Grafana生成的多值时间序列图截图。这个图表显示了过去30天'AAPL'的股价为绿色，'AMD'的股价为黄色。](https://assets.timescale.com/docs/images/tutorials/visualizations/time-series/multi-value-graph.png)

</Procedure>

[time_bucket]: /api/:currentVersion:/hyperfunctions/time_bucket/

