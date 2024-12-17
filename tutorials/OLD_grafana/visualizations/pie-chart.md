---
标题: 在格拉法纳（Grafana）中构建饼图
摘要: 在格拉法纳（Grafana）中绘制饼图，以对比不同类别间的值。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析]
标签: [饼图]
---

import GrafanaVizPrereqs from 'versionContent/_partials/_grafana-viz-prereqs.mdx';

# 在Grafana中构建饼图

饼图用于绘制分类数据。图表将每个类别呈现为饼图的一块，以便您可以看到它对总数的贡献。需要注意的是，随着切片数量的增加，图表变得更加难以分析，且如果有很多相似的小切片，比较切片大小也变得更加困难。对于少量类别使用饼图，并在有大量数据时考虑使用其他图表类型。

饼图可以回答以下问题：

*   上个月交易量最少的股票是哪一只？
*   今天交易量最高的股票是哪一只？
*   上一次选举中累积票数百分比最高的是谁？

本教程向您展示如何：

*   使用`time_bucket()`创建预聚合数据的饼图。
*   创建一个甜甜圈图以显示股票交易量的分布。

饼图可以是传统的饼图样式，也可以是甜甜圈样式。两者显示相同的信息。本教程向您展示如何创建这两种图表。

## 前提条件

<GrafanaVizPrereqs />

## 使用预聚合数据创建饼图

使用表`stocks_real_time`中的数据创建饼图可视化。

<Procedure>

### 使用预聚合数据创建饼图

1.  在查询编辑器中，使用以下SQL查询饼图数据集。使用变量`$bucket_interval`表示饼图覆盖的时间周期：

    ```sql
    SELECT
        time_bucket('$bucket_interval', time) AS time,
        symbol,
        AVG(price) AS price
    FROM stocks_real_time srt
    WHERE symbol IN ($symbol)
        AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
    GROUP BY time_bucket('$bucket_interval', time), symbol
    ORDER BY time_bucket('$bucket_interval', time), symbol;
    ```

2.  使用以下查询从[入门教程][gsg-data]的数据集中获取所有公司符号：

    ```sql
    SELECT
        DISTINCT symbol FROM company ORDER BY symbol ASC;
    ```

3.  在Grafana仪表板中，导航到您的面板的设置页面。在`Variables`部分：

    *   在`Name`字段中，为您的符号命名。
    *   在`Type`字段中，选择`Query`。
    *   在`Selection options`字段中，启用`multi-value`。

4.  在Grafana仪表板中，在`Dashboard variable`字段中，选择要绘制的股票。根据需要调整仪表板的时间范围。确保返回的数据有一个名为`time`的列，包含时间戳。时间戳应按升序排列。否则，您将获得一个错误。返回的数据如下所示：

    ![有效的时间序列数据表视图截图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/piechart/piecharttabledata2.png)

5.  在`Visualizations`字段中，选择`Pie chart`。Grafana将查询转换为饼图。此示例显示了JPM、IBM、AAPL、AMD和CVS股票的价格分布饼图，分别为20%、23%、25%、15%和17%，在特定时期内。返回的数据包含大量信息，但由于计算字段中选择的选项，仅显示了每只股票的首值。

    ![Grafana生成的饼图截图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/piechart/piecharttable2view.png)

6.  您可以通过更改`Show`的值选项来更改数据的显示方式。上一步中使用的`Calculate`选项将每个时间序列简化为单个值。`All values`选项显示每个系列的每个值。选择`All values`以查看差异。

    ![Grafana生成的饼图中显示的所有值的截图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/piechart/datadisplaytype.png)

7.  新图表如下所示。请注意，每只股票都有多个切片来表示它，对应于表中的每行。切片的数量限制为40：

   ![Grafana生成的饼图截图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/piechart/piechart2.png)

</Procedure>

## 创建带有交易量的甜甜圈图

甜甜圈图是中间挖空的饼图。与饼图相比，甜甜圈图上的段更容易比较，因为人眼更擅长比较弧线的长度。否则，它们是完全相同的。

按照本节创建一个甜甜圈图，显示每个股票在时间桶内的平均交易量。每个桶的交易量是根据`stocks_real_time`超表中提供的每日累积交易量计算得出的。

<Procedure>

### 创建带有交易量的甜甜圈图

1.  在psql提示符下，更新之前的查询，以找到桶内符号的最大`day_volume`值。然后，从上一个桶的最大值中减去每个最大值。差值给出了该桶的交易量：

    ```sql
      SELECT
          time_bucket('$bucket_interval', time) AS time,
          symbol,
          MAX(day_volume) - LAG(MAX(day_volume), 1) OVER(
              PARTITION BY symbol
              ORDER BY time_bucket('$bucket_interval', time)
          ) AS bucket_volume
      FROM stocks_real_time srt
      WHERE symbol = $symbol
          AND time >= $__timeFrom()::timestamptz AND time < $__timeTo()::timestamptz
      GROUP BY time_bucket('$bucket_interval', time), symbol;
      ORDER BY time_bucket('$bucket_interval', time), symbol;
      ```

2.  在Grafana仪表板中，将您的饼图转换为甜甜圈图。在符号下拉菜单中，选择您想要比较的所有股票。在面板的右侧，点击饼图下拉菜单。在`piechart type`字段中，选择`donut`：

  ![Grafana仪表板截图，显示饼图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/piechart/piecharttype.png)

3.  刷新面板。甜甜圈图视图显示了整个一天内10分钟桶的交易量百分比：

   ![Grafana仪表板截图，显示甜甜圈图](https://s3.amazonaws.com/assets.timescale.com/docs/images/tutorials/visualizations/piechart/donutchart.png)

<Highlight type="note">
如果您查看超过一个交易日的数据，您可能会得到看起来不太好的结果，或者您可能没有数据返回。要解决这个问题，请将计算集中在单个交易日上。
</Highlight>

</Procedure>

饼图是用于比较分类数据的一个很好的工具。它们特别适合于可视化百分比。但如果您有太多类别的百分比相似或大量数据，它们的效果就不太好。

[gsg-data]: https://docs.timescale.com/getting-started/latest/time-series-data/

