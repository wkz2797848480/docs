---
标题: 绘制地理空间时间序列数据教程
摘要: 学习如何绘制地理空间时间序列数据。
产品: [云服务]
关键词: [教程，地理信息系统（GIS），地理空间，学习]
标签: [教程，中级]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 绘制纽约市出租车地理空间数据
---

# 绘制地理空间的纽约出租车数据

纽约市大约有900万人口。本教程使用由纽约市出租车和豪华轿车委员会 [NYC TLC][nyc-tlc] 提供的纽约黄色出租车网络的历史数据。NYC TLC跟踪超过20万辆车辆，每天大约进行100万次行程。由于这些数据几乎全是时间序列数据，适当的分析需要一个专门构建的时间序列数据库，如Timescale。

在[初学者纽约出租车教程][beginner-fleet]中，您研究了构建查询，查看了乘坐次数以及时间。纽约出租车数据集还包含了每次行程的接送地点信息。这是地理空间数据，您可以使用一个名为PostGIS的PostgreSQL扩展来检查行程的起始地点。此外，您可以通过在Grafana中将其叠加在地图上来可视化数据。

## 前提条件

开始之前，请确保您已具备以下条件：

*   注册了[免费Timescale账户][cloud-install]。
*   [](#)<Optional /> 如果您想要绘制查询图表，注册了[Grafana账户][grafana-setup]。

## 本教程的步骤

本教程包括：

1.  [设置您的数据集][dataset-nyc]：设置并连接到Timescale服务，并使用`psql`将数据加载到您的数据库中。
2.  [查询您的数据集][query-nyc]：使用Timescale和PostgreSQL分析包含纽约出租车行程数据的数据集，并在Grafana中绘制结果。

## 关于使用Timescale查询数据

本教程使用[纽约出租车数据][nyc-tlc]向您展示如何构建地理空间时间序列数据的查询。您在本教程中进行的分析与公民组织规划新道路和公共服务的分析类似。

它首先教您如何设置和连接到Timescale数据库，创建表，并使用`psql`将数据加载到表中。如果您已经完成了[第一个纽约出租车教程][beginner-fleet]，那么您已经加载了数据集，您可以跳过[直接进行查询][plot-nyc]。

然后，您将学习如何对您的数据集进行分析和监控。它指导您使用带有PostGIS扩展的PostgreSQL查询来获取信息，并在Grafana中绘制结果。

[dataset-nyc]: /tutorials/:currentVersion:/nyc-taxi-geospatial/dataset-nyc/
[query-nyc]: /tutorials/:currentVersion:/nyc-taxi-geospatial/plot-nyc/
[nyc-tlc]: https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page 
[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account
[beginner-fleet]: /tutorials/:currentVersion:/nyc-taxi-cab/
[plot-nyc]: /tutorials/:currentVersion:/nyc-taxi-geospatial/plot-nyc/
[grafana-setup]: /use-timescale/:currentVersion:/integrations/observability-alerting/grafana/installation/


