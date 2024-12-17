---
标题: 能源消耗数据教程
摘要: 学习如何查询能源消耗数据。
产品: [云服务]
关键词: [教程，能源，学习]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析能源消耗数据
---

# 分析能耗数据

当您计划转换为屋顶太阳能系统时，即使有专家在手，这也不是一件容易的事。您需要了解您的电力消耗详情、典型使用小时或一年中的分布情况。收集几秒钟粒度的消耗数据是找到更精确答案的关键。本教程使用了一个典型家庭一年多的能耗数据。因为几乎所有这些数据都是时间序列数据，适当的分析需要一个专门的时间序列数据库，如Timescale。

在这个教程中，您可以构建查询来查看消耗了多少瓦特电量，以及何时消耗。此外，您还可以在Grafana中可视化能耗数据。

## 前提条件

开始之前，请确保您已经：

*   注册了[免费的Timescale账户][cloud-install]。
*   []()<Optional />注册了[Grafana账户][grafana-setup]来绘制查询结果。

## 本教程的步骤

本教程涵盖：

1.  [设置您的数据集][dataset-energy]：设置并连接到Timescale服务，并使用`psql`将数据加载到数据库中。
2.  [查询您的数据集][query-energy]：使用Timescale和PostgreSQL分析包含能耗数据的数据集，并在Grafana中可视化结果。
3.  [附加内容：高效存储数据][compress-energy]：学习如何使用Timescale的压缩功能更高效地存储和查询您的能耗数据。

## 关于使用Timescale查询数据

本教程使用样本能耗数据向您展示如何构建时间序列数据的查询。您在本教程中进行的分析类似于家庭可能用来规划他们的太阳能安装或优化随时间的能源使用的分析类型。

它首先教您如何设置并连接到Timescale数据库，创建表，并使用`psql`将数据加载到表中。

然后，您将学习如何对您的数据集进行分析和监控。它还引导您了解如何在Grafana中可视化结果的步骤。

[dataset-energy]: /tutorials/:currentVersion:/energy-data/dataset-energy/
[query-energy]: /tutorials/:currentVersion:/energy-data/query-energy/
[compress-energy]: /tutorials/:currentVersion:/energy-data/compress-energy/
[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account
[grafana-setup]: /use-timescale/:currentVersion:/integrations/observability-alerting/grafana/installation/
