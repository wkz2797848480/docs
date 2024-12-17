---
标题: 使用格拉法纳（Grafana）可视化存储于 TimescaleDB 的地理空间数据
摘要: 利用世界地图可视化功能在世界地图上查看地理空间数据的分布情况。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [格拉法纳（Grafana），可视化，分析，地理空间数据]
---

# 使用Grafana可视化TimescaleDB中存储的地理空间数据

Grafana包含了一个WorldMap可视化工具，可以帮助您在世界地图上叠加地理空间数据。这对于理解基于位置变化的数据非常有用。

### 前提条件

完成本教程，您需要对结构化查询语言（SQL）有初步了解。教程将指导您完成每个SQL命令，但如果您之前接触过SQL将会有所帮助。

*   首先，[安装TimescaleDB][install-timescale]。
*   接下来，设置Grafana。

当您的TimescaleDB和Grafana安装完成后，摄取纽约出租车教程中的数据，并将Grafana配置为连接到该数据库。如果您对如何使用TimescaleDB感兴趣，请确保按照完整的教程操作。

<Highlight type="tip">
确保密切关注教程中的地理空间查询部分，并完成这些步骤。

</Highlight>

### 构建地理空间查询

纽约出租车数据还包含了每次行程的接客地点。在纽约出租车教程中，我们检查了从时代广场附近出发的行程。让我们在该查询的基础上**可视化在曼哈顿行程距离超过五英里的行程**。

我们可以使用Grafana的“Worldmap Panel”来实现这一点。首先创建一个新的面板，选择“New Visualization”，然后选择“Worldmap Panel”。

再次，您可以直接编辑查询。在查询屏幕上，确保选择您的纽约出租车数据作为数据源。在“Format as”下拉菜单中，选择“Table”。点击“Edit SQL”并在文本窗口中输入以下查询：

```sql
SELECT time_bucket('5m', rides.pickup_datetime) AS time,
       rides.trip_distance AS value,
       rides.pickup_latitude AS latitude,
       rides.pickup_longitude AS longitude
FROM rides
WHERE $__timeFilter(rides.pickup_datetime) AND
  ST_Distance(pickup_geom,
              ST_Transform(ST_SetSRID(ST_MakePoint(-73.9851,40.7589),4326),2163)
  ) < 2000
GROUP BY time,
         rides.trip_distance,
         rides.pickup_latitude,
         rides.pickup_longitude
ORDER BY time
LIMIT 500;
```

让我们解析这个查询。首先，我们希望用视觉标记来绘制行程，表示行程距离。较长距离的行程在地图上有不同的视觉处理。使用`trip_distance`作为我们图表的值，并将其存储在`value`字段中。

在`SELECT`语句的第二行和第三行中，我们使用数据库中的`pickup_longitude`和`pickup_latitude`字段，并将它们映射到变量`longitude`和`latitude`。

在`WHERE`子句中，我们应用地理空间边界，寻找在时代广场2000米范围内的行程。

最后，在`GROUP BY`子句中，我们提供`trip_distance`和位置变量，以便Grafana能够正确绘制数据。

<Highlight type="warning">
这个查询可能需要一些时间，取决于您的互联网连接速度。这就是为什么我们为了演示目的使用`LIMIT`语句。

</Highlight>

### 配置Worldmap Grafana面板

现在让我们配置我们的Worldmap可视化。在Grafana用户界面的最左侧选择“Visualization”标签。您将看到“Map Visual Options”、“Map Data Options”等选项。

首先，确保“Map Data Options”设置为“table”和“current”。然后在“Field Mappings”部分，将“Table Query Format”设置为“Table”。我们可以将“Latitude Field”映射到我们的`latitude`变量，将“Longitude Field”映射到`longitude`变量，将“Metric”字段映射到`value`变量。

在“Map Visual Options”中，将“Min Circle Size”设置为1，将“Max Circle Size”设置为5。

在“Threshold Options”中，将“Thresholds”设置为“2,5,10”。这自动配置了一组颜色。任何`value`低于2的图表是一种颜色，任何`value`在2到5之间的图表是另一种颜色，任何`value`在5到10之间的图表是第三种颜色，任何`value`超过10的图表是第四种颜色。

您的配置应该如下所示：

![在Grafana中将Worldmap字段映射到查询结果](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana-fieldmapping.png)

此时，数据应该流入我们的Worldmap可视化，如下所示：

![使用Grafana Worldmap可视化PostgreSQL中的时间序列数据](https://assets.iobeam.com/images/docs/screenshots-for-grafana-tutorial/grafana_worldmap_query_results.png)

您应该能够编辑可视化顶部的时间过滤器，以查看不同时间段的行程接客数据。

### 总结

通过遵循所有TimescaleDB + Grafana教程，完成您的Grafana知识。

[install-timescale]: /getting-started/latest/

