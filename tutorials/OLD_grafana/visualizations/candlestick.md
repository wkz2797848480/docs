---
标题: 在格拉法纳（Grafana）中构建 K 线图
摘要: 在格拉法纳（Grafana）中创建 K 线图，以可视化呈现金融资产的开盘价、收盘价、最高价和最低价。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析，金融]
标签: [K 线]
搜索引擎优化（SEO）: 
元图像: https://s3.amazonaws.com/assets.timescale.com/docs/images/meta-images/meta-image-grafana-candlestick.png
---

import GrafanaVizPrereqs from 'versionContent/_partials/_grafana-viz-prereqs.mdx';

# 在Grafana中构建K线图

K线图显示了金融资产如股票、货币和证券的开盘价、收盘价、最高价和最低价。它们主要用于技术分析，以预测价格将如何变化。

它们可以回答以下问题：

*   这一天的资产开盘价、收盘价、最高价和最低价是多少？
*   这段时间内开盘价和收盘价之间的价差是多少？
*   这个资产的价格随时间如何变化？
*   这个资产是否正在进入熊市或牛市？

![K线图](https://assets.timescale.com/docs/images/tutorials/visualizations/candlestick/candlestick_fig.png) 

上图显示了K线的结构。一个K线覆盖一个特定的时间间隔，例如5分钟、10分钟或1小时。对于这个时间段，它绘制了四个值：

*   **开盘价**：起始价格
*   **收盘价**：结束价格
*   **最高价**：最高价格
*   **最低价**：最低价格

K线图可以在时间上显示多个K线。这有助于您看到资产价格变化的模式。例如，您可以判断一个资产是否正在进入牛市或熊市，或者其市场活动是否达到顶峰或低谷。

本教程向您展示如何：

*   使用原始数据创建K线聚合。
*   在查询原始数据时显示交易量。

## 前提条件

<GrafanaVizPrereqs />

查看这个视频，了解如何在Grafana中创建K线图的逐步指导：
<Video url="https://www.youtube-nocookie.com/embed/08CydeL9lIk"/>

## 使用原始数据创建K线图

使用表`stocks_real_time`中的原始数据创建K线图可视化。

<Procedure>

### 使用原始数据创建K线图

1.  在查询编辑器中，使用以下SQL查询K线数据集。使用变量`$bucket_interval`表示每个K线覆盖的时间周期。

    ```sql
    SELECT
        time_bucket('$bucket_interval', time) AS time,
        symbol,
        FIRST(price, time) AS "open",
        MAX(price) AS high,
        MIN(price) AS low,
        LAST(price, time) AS "close"
    FROM stocks_real_time
    WHERE symbol IN ($symbol)
        AND time > $__timeFrom()::timestamptz and time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time), symbol;
    ```

2.  点击查询编辑器外部，或点击刷新图标以更新Grafana图表。

3.  选择`candlestick`作为您的可视化类型：

    ![Grafana中的K线图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/candlestick/candlestick_visualization.png)

4.  Grafana将查询转换为K线图，如下所示：

    ![Grafana中的K线图](https://assets.timescale.com/docs/images/tutorials/visualizations/candlestick/1_min.png)

    在这个第一个例子中，`$bucket_interval`设置为1分钟，您可以看到`AMZN`的价格在2120美元到2200美元之间变化。这个图表使用超函数查询`stock_real_time`表，时间间隔为1分钟。

</Procedure>

检索这些数据大约需要7秒以上，覆盖两周的数据，这可能比大多数用户在分析数据时预期的要慢。这时，连续聚合对于数据密集型的时间序列应用特别有用。

![Grafana查询响应](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/candlestick/raw_data_exec_time.png)

<Procedure>

当您使用`$bucket_interval`变量时，可以切换时间间隔。例如，切换到15分钟的时间间隔，您将得到一个这样的数据。

1.  从下拉菜单中将时间间隔切换到15分钟：

    ![Grafana变量下拉菜单](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/candlestick/timebucket_dropdown.png)

2.  刷新仪表板以获取更新后的图表：

    ![Grafana中的K线图](https://assets.timescale.com/docs/images/tutorials/visualizations/candlestick/15_min.png)

查询执行时间超过6秒。要将查询执行时间减少到亚秒级别，请使用连续聚合。有关更多信息，请参见[连续聚合][continuous-aggregrate]的指南。

</Procedure>

## 在K线图中显示交易量

除了查看每只股票的价格变化外，您还可以查看其交易量。这可以显示在时间间隔内股票的交易量。

`stock_real_time`超表包含一个列，用于记录每日累积交易量。您可以使用此列计算每个时间桶的数据量。

首先，在时间桶内找到符号的最大`day_volume`值。然后，从上一个桶的最大值中减去每个最大值。差值给出了该桶的交易量。

<Procedure>

### 在K线图中显示交易量

1.  使用以下查询创建一个新的K线面板：

    ```sql
    SELECT
        time_bucket('$bucket_interval', time) AS time,
        symbol,
        FIRST(price, time) AS "open",
        MAX(price) AS high,
        MIN(price) AS low,
        LAST(price, time) AS "close",
        MAX(day_volume) - LAG(max(day_volume), 1) OVER(
          PARTITION BY symbol
          ORDER BY time_bucket('$bucket_interval', time)
        ) AS bucket_volume
    FROM stocks_real_time
    WHERE symbol IN ($symbol)
        AND time > $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time), symbol;
    ```

2.  刷新仪表板以获取更新后的图表。

    ![Grafana中的K线图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/candlestick/volume_Distribution.png)

    在图表的底部，您可以看到每个时间桶的交易量。

</Procedure>

总之，K线图是可视化金融数据的好方法。本教程向您展示了如何使用TimescaleDB从超表中的原始数据生成包含开盘、最高、最低和收盘价的K线值。它还向您展示了如何查询每个时间间隔的交易量。

[continuous-aggregrate]: /tutorials/:currentVersion:/financial-tick-data/

