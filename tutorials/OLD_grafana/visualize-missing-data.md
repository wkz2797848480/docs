---
标题: 如何在格拉法纳（Grafana）中可视化并聚合缺失的时间序列数据
摘要: 在格拉法纳（Grafana）绘制数据时处理缺失的数据点。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析，间隙填充]
标签: [时间序列]
---

# 教程：如何在Grafana中可视化和聚合缺失的时间序列数据

### 引言

有时我们的时间序列数据中存在间隙：因为系统离线或设备失去电源等。这在您想要聚合大时间窗口中的数据时会导致问题，例如，计算过去6小时的平均温度，每30分钟间隔一次，或分析今天的CPU利用率，每15分钟间隔一次。数据中的间隙还可能产生其他负面影响，例如破坏下游应用程序。

在本教程中，您将了解如何使用[Grafana][grafana-external]（一个开源可视化工具）和TimescaleDB来处理缺失的时间序列数据（使用Grafana中原生可用的TimescaleDB/PostgreSQL数据源）。

### 前提条件

要完成本教程，您需要对结构化查询语言（SQL）有初步了解。教程将指导您完成每个SQL命令，但如果您之前接触过SQL将会有所帮助。

您还需要：

*   带有缺失数据的时间序列数据集（注意：如果您手头没有，我们在下面提供了创建一个的可选步骤。）

*   一个正在运行的[TimescaleDB安装][install-timescale]。安装完成后，我们可以继续摄取或创建样本数据并完成教程。

*   连接到您的TimescaleDB实例的Grafana仪表板。

### 第0步 - 将您的时间序列数据加载到TimescaleDB并模拟缺失数据（可选）

*（如果您已经有TimescaleDB加载了您的时间序列数据，请跳过此步骤。）*

对于本教程，我们将加载我们的TimescaleDB实例与模拟的IoT传感器数据（可在我们如何使用模拟的IoT传感器数据探索TimescaleDB教程中找到）。

此数据集模拟了四个传感器，每个传感器收集温度和CPU数据，组织在如下的[超表][docs-hypertable]中：

```sql
CREATE TABLE sensor_data (
  time TIMESTAMPTZ NOT NULL,
  sensor_id INTEGER,
  temperature DOUBLE PRECISION,
  cpu DOUBLE PRECISION,
  FOREIGN KEY (sensor_id) REFERENCES sensors (id)
);
```

为了模拟缺失数据，让我们删除我们的传感器在1小时前和2小时前收集的所有数据：

```
DELETE FROM sensor_data WHERE sensor_id = 1 and time > now() - INTERVAL '2 hour' and time < now() - INTERVAL '1 hour';
```

### 第1步 - 绘制数据集并确认缺失数据

*（对于这一步和后续步骤，我们将使用第0步中的IoT数据集，但如果您使用自己的-真实或模拟的-数据集，步骤相同）*

为了确认我们缺少数据值，让我们创建一个简单的图表，计算过去6小时`sensor_1`的平均温度读数（使用[`time_bucket`][docs-timebucket]）。

```sql
SELECT
  time_bucket('5 minutes', "time") as time,
  AVG(temperature) AS sensor_1
FROM sensor_data
WHERE
  $__timeFilter("time") AND
  sensor_id = 1
GROUP BY time_bucket('5 minutes', time)
ORDER BY 1
```

![Grafana截图：缺失数据](https://assets.iobeam.com/images/docs/screenshots-for-tutorial-missing-data-grafana/missing-data.png)

我们可以看到，在17:05到18:10之间缺少数据，因为那段时间内缺少数据点（平直线）。

### 第2步 - 插值（填充）缺失数据

为了插值缺失数据，我们使用[`time_bucket_gapfill`][docs-timebucket-gapfill]，结合[`LOCF`][docs-LOCF]（“Last Observation Carried Forward”）。这会取缺失数据开始前的最后一个读数，并在规律的时间间隔中绘制它（最后一个记录的值），直到接收到新数据：

```sql
SELECT
  time_bucket_gapfill('5 minutes', "time") as time,
  LOCF(AVG(temperature)) AS sensor_1
FROM sensor_data
WHERE
  $__timeFilter("time") AND
  sensor_id = 1
GROUP BY time_bucket_gapfill('5 minutes', "time")
ORDER BY 1
```

LOCF是处理缺失数据时的便捷插值技术，但您没有额外的上下文来确定缺失数据值可能是什么。

![Grafana截图：使用LOCF插值](https://assets.iobeam.com/images/docs/screenshots-for-tutorial-missing-data-grafana/locf.png)

如您所见，图表现在在规律的时间间隔中绘制数据点，填补了我们缺失数据的时间。

### 第3步 - 在更大的时间窗口中聚合

现在，我们回到最初的问题：希望在存在缺失数据的大时间窗口中聚合数据。

这里我们使用插值数据，并计算过去6小时每30分钟窗口的平均温度。

```sql
SELECT
  time_bucket_gapfill('30 minutes', "time") as time,
  LOCF(AVG(temperature)) AS sensor_1
FROM sensor_data
WHERE
  $__timeFilter("time") AND
  sensor_id = 1
GROUP BY time_bucket_gapfill('30 minutes', "time")
ORDER BY 1
```

![Grafana截图：跨插值数据聚合](https://assets.iobeam.com/images/docs/screenshots-for-tutorial-missing-data-grafana/aggregate.png)

让我们通过在图表中添加一个新的系列来比较，如果我们没有插值缺失数据，聚合会是什么样子：

```sql
SELECT
  time_bucket('30 minutes', "time") as time,
  AVG(temperature) AS sensor_1
FROM sensor_data
WHERE
  $__timeFilter("time") AND
  sensor_id = 1
GROUP BY time_bucket('30 minutes', time)
ORDER BY 1
```

![Grafana截图：跨插值数据与缺失数据聚合对比](https://assets.iobeam.com/images/docs/screenshots-for-tutorial-missing-data-grafana/aggregate_2.png)

（请注意，插值平均值现在是橙色，而带有缺失数据的平均值是绿色。）

如您所见，绿色图表在17:30缺少一个数据点，这让我们对那段时间内发生的事情知之甚少，可能会破坏下游应用程序。相比之下，橙色图表使用我们的插值数据为那个时间段创建了一个数据点。

### 后续步骤

这只是使用TimescaleDB与Grafana解决数据问题，确保您的应用程序、系统和操作不会遭受任何负面影响（例如，停机时间、行为异常的应用程序或降低的顾客体验）的一种方式。要了解更多使用TimescaleDB的方法，请查看我们的其他[教程][tutorials]（从初级到高级）。

[docs-LOCF]: /api/:currentVersion:/hyperfunctions/gapfilling/time_bucket_gapfill#locf
[docs-hypertable]: /use-timescale/:currentVersion:/hypertables/
[docs-timebucket-gapfill]: /api/:currentVersion:/hyperfunctions/gapfilling/time_bucket_gapfill/
[docs-timebucket]: /api/:currentVersion:/hyperfunctions/time_bucket
[grafana-external]: https://grafana.com/ 
[install-timescale]: /getting-started/latest/
[tutorials]: /tutorials/:currentVersion:/

