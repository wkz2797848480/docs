---
标题: 在格拉法纳（Grafana）中构建直方图
摘要: 在格拉法纳（Grafana）中创建直方图，以可视化呈现数据的分布情况。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析]
标签: [直方图]
---

import GrafanaVizPrereqs from 'versionContent/_partials/_grafana-viz-prereqs.mdx';

# 在Grafana中构建直方图

直方图显示数据的分布情况。您可以使用直方图来绘制在某个尺度上的桶中落入的数据点数量。例如，直方图通常用于显示金融工具的分布情况。

它们可以回答以下问题：

*   Meta股票今天的价格分布是怎样的？
*   上周AMD股票的交易量分布如何？
*   过去一年中标准普尔500指数的日回报分布如何？

## Grafana直方图的数据

使用Grafana，您可以通过提供3种格式之一的数据来绘制直方图。每种方法都有其自身的优势和挑战：

*   **原始数据**：这种方法不需要您预先对数据进行桶处理或预聚合。
  它提高了直方图的准确性。但它需要更多的CPU、内存和网络使用量，因为所有的桶处理都是在浏览器中完成的。这可能会导致严重的性能问题。
*   **预桶数据**：您需要配置数据源来预先对您的数据进行桶处理。根据Grafana文档，任何源都可以为直方图输出预桶数据，只要它满足数据格式要求。例如，它建议您可以使用Elastic Search的直方图桶聚合或Prometheus的直方图指标。
*   **聚合数据**：Grafana也接受预聚合的时间桶数据。您可以使用TimescaleDB的`time_bucket`函数或PostgreSQL的`date_trunc`函数来聚合您的数据。要创建直方图，Grafana会进一步对聚合数据进行桶处理。它自动选择桶大小，大约为您数据总范围的10%。

<Highlight type="note">
直方图非常适合分析数据的分布或扩散情况，但它们不显示数据随时间的变化。如果您需要查看数据随时间的分布，请尝试使用热图代替。
</Highlight>

## 您将学到的内容

本教程向您展示如何：

*   从原始数据创建价格/交易直方图
*   从预聚合数据创建价格/交易直方图
*   创建显示多个直方图的面板
*   创建价格/成交量直方图

## 前提条件

<GrafanaVizPrereqs />

查看这个视频，了解在Grafana中创建直方图的逐步操作：
<Video url="https://www.youtube-nocookie.com/embed/h1eTIYOFplA&#34;/&gt; 

## 使用原始数据创建价格/交易直方图

评估股票交易数据的常见直方图是价格/交易量直方图。这显示了在给定价格范围内发生的交易数量，在一个时间间隔内。要制作这个直方图，请从`stocks_real_time`超表中选择原始交易数据。

<Procedure>

### 使用原始数据创建价格/交易直方图

1.  在Grafana仪表板中添加此查询：

    ```sql
    SELECT time,
        price
    FROM stocks_real_time srt
    WHERE symbol = '$symbol'
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    ORDER BY time;
    ```

2.  在仪表板变量中选择一个股票。如需要，调整仪表板的时间范围。

3.  返回的数据如下所示：

    ```bash
    time                         |price   |
    -----------------------------+--------+
    2022-03-02 17:01:07.000 -0700|  166.33|
    2022-03-02 17:01:26.000 -0700|165.8799|
    2022-03-02 17:01:31.000 -0700|165.8799|
    2022-03-02 17:01:46.000 -0700|  166.43|
    2022-03-02 17:02:22.000 -0700|  166.49|
    2022-03-02 17:02:40.000 -0700|166.6001|
                 …               |   …    |
    ```

    任何时间序列数据的关键特征是，它必须有一个名为`time`的列，包含时间戳数据。用于绘制数据的其他列可以有不同的名称，但每个时间序列图表都必须在结果中有`time`列。对于直方图可视化，时间戳值必须按升序排列，否则您将收到错误。

4.  选择“Histogram”作为您的可视化类型。

    ![Grafana仪表板截图。“Visualizations”标签被选中。在下面，“Histogram”显示为可视化类型。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/histogram_panel_selection.png)

5.  Grafana将查询转换为直方图，如下所示：

    ![Grafana中的直方图截图，显示了$AAPL的价格分布。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/simple_histogram.png)

    直方图显示$AAPL的价格在$154到$176之间。Grafana自动为我们选择了桶大小，在这个例子中是$2。

6.  要增加直方图的粒度，将桶大小从2更改为0.1。

    ![Grafana中直方图下拉菜单的截图。“bucket size”输入字段的值为0.1。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/bucket_size_option.png)

7.  直方图现在看起来类似，但显示了更多的细节。

    ![Grafana中的直方图截图，显示了$AAPL的价格分布，桶大小为$1。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/simple_histogram_with_bucket_size.png)

</Procedure>

## 使用聚合数据创建价格/交易直方图

之前的示例查询了苹果股票的原始数据，该股票通常每天交易约40,000次。查询返回了超过40,000行数据供Grafana在每个刷新间隔（默认为30秒）进行桶处理，这使用了大量的CPU、内存和网络带宽。在极端情况下，Grafana会显示消息：
`Results have been limited to 1000000 because the SQL row limit was reached`

这意味着Grafana没有显示查询返回的所有行。要解决这个问题，请使用TimescaleDB的`time_bucket`函数在查询中预先聚合数据。使用`time_bucket`时，您需要添加一个名为`bucket_interval`的新变量。

<Procedure>

### 使用预聚合数据创建价格/交易直方图

1.  在Grafana中，添加一个名为`$bucket_interval`的新变量，类型为`INTERVAL`。

    <!-- vale Google.Units = NO -->

    ![Grafana截图，显示“Variables > Edit”对话框。变量名为'bucket_interval'，类型为“interval”，并给出了从10s到30d的间隔选项值。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/bucket_interval_variable_options.png)

    <!-- vale Google.Units = YES -->

2.  使用`$bucket_interval`变量来聚合选定间隔的价格：

    ```sql
    SELECT time_bucket('$bucket_interval', time) AS time,
        AVG(price) avg_price
    FROM stocks_real_time srt
    WHERE symbol = '$symbol'
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time);
    ```

3.  此查询产生的直方图如下所示：

    ![Grafana中的直方图截图，显示了$AAPL的价格分布。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/time_bucket_histogram.png)

    这个第二个直方图看起来与使用原始交易数据的示例相似。但查询只返回了大约1000行数据，每次请求。这减少了网络负载和Grafana处理时间。

</Procedure>

## 创建带有多个价格/交易直方图的面板

要比较多个不同股票的分布，创建一个带有多个直方图的面板。将`$symbol`变量从文本变量更改为查询变量，并启用多值选项。这允许您为`$symbol`选择多个值。数据库返回所有选定值的交易，Grafana将它们分别放入不同的直方图中。

<Procedure>

### 创建带有多个价格/交易直方图的面板

1.  从数据集中获取所有公司符号：

    ```sql
    SELECT DISTINCT symbol FROM company ORDER BY symbol ASC;
    ```

2.  使用先前的查询创建一个新的仪表板变量：

    ![Grafana截图，显示“Variables > Edit”对话框。变量名为'symbol'，类型为'Query'。查询输入字段已用第一步中的查询填写。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/symbol_variable_query.png)

3.  更新主查询如下：

    ```sql
    SELECT time_bucket('$bucket_interval', time) AS time,
        symbol,
        AVG(price) AS avg_price
    FROM stocks_real_time srt
    WHERE symbol IN
 ($symbol)
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time), symbol
    ORDER BY time;
    ```

4.  此查询结果产生以下直方图：

    ![Grafana中叠加直方图的截图，显示了3种股票的价格分布。所有3个直方图都是绿色的。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/multiple_histograms.png)

    您可以清楚地看到3个不同的直方图，但它们之间无法区分。

5.  点击图例左侧的绿色线条并选择颜色：

    ![Grafana颜色选择器的截图。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/color_options.png)

6.  图表清楚地显示了3种价格分布的不同颜色：

    ![Grafana图表截图，显示了3个直方图的股票值分别为蓝色、红色和绿色。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/colored_in_histogram.png)

</Procedure>

## 创建价格/成交量直方图

除了交易价格，您还可以查看交易量。交易量的分布显示了人们购买股票的频率和数量。

`stocks_real_time`超表包含一个列，记录每日累计成交量。您可以使用这个来计算每个桶的数据量。首先，找到桶内符号的最大`day_volume`值。然后从上一个桶的最大值中减去每个最大值。差值等于该桶的成交量。

您可以使用预聚合查询来实现这一点，使用：

*   TimescaleDB的[`time_bucket`][time_bucket]函数。
*   PostgreSQL的[`max`][max]函数。
*   PostgreSQL的[`lag`][lag]函数。使用这个来减去每个上一个桶的最大值，当行按降序`time`排序时。

<Procedure>

### 创建价格/成交量直方图

1.  使用以下查询创建一个新的直方图面板：

    ```sql
    WITH buckets AS (
        SELECT time_bucket('$bucket_interval', time) AS time,
            symbol,
            MAX(day_volume) AS dv_max
        FROM stocks_real_time
        WHERE time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
            AND day_volume IS NOT NULL
            AND symbol IN ($symbol)
        GROUP BY time_bucket('$bucket_interval', time), symbol
    )
    SELECT TIME,
        symbol,
        CASE WHEN lag(dv_max ,1) OVER (PARTITION BY symbol ORDER BY time) IS NULL THEN dv_max
        WHEN (dv_max - lag(dv_max, 1) OVER (PARTITION BY symbol ORDER BY time)) < 0 THEN dv_max
        ELSE (dv_max - lag(dv_max, 1) OVER (PARTITION BY symbol ORDER BY time)) END vol
    FROM buckets
    ORDER BY time;
    ```

2.  此查询结果产生以下直方图：

    ![Grafana直方图截图，显示了$AMZN的股票成交量分布。](https://assets.timescale.com/docs/images/tutorials/visualizations/histograms/volume_distribution.png)

    图表显示了AMZN符号的左偏分布。对于许多符号，您可能会看到由于几个非常大的成交量交易而扭曲的分布。有两种解决方案：
    *   将查询限制在成交量小于某个阈值的交易。
    *   使用对数刻度。不幸的是，Grafana直方图面板尚不支持这些。

</Procedure>



[lag]: <https://www.postgresql.org/docs/14/functions-window.html>
[max]: <https://www.postgresql.org/docs/current/tutorial-agg.html>
[time_bucket]: /api/:currentVersion:/hyperfunctions/time_bucket/
