---
标题: 查询时间序列数据教程
摘要: 学习如何查询时间序列数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，查询，学习]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析纽约市出租车数据
---

import PreloadedData from "versionContent/_partials/_preloaded-data.mdx";

# 分析纽约出租车数据

纽约市大约有900万人口。本教程使用由纽约市出租车和豪华轿车委员会[NYC TLC][nyc-tlc]提供的纽约黄色出租车网络的历史数据。NYC TLC跟踪超过20万辆车辆，每天大约进行100万次行程。由于这些数据几乎全是时间序列数据，适当的分析需要一个专门构建的时间序列数据库，如Timescale。

## 前提条件

开始之前，请确保您已具备以下条件：

*   注册了[免费Timescale账户][cloud-install]。

## 本教程的步骤

本教程包括：

1.  [设置您的数据集][dataset-nyc]：设置并连接到Timescale服务，并使用`psql`将数据加载到您的数据库中。
2.  [查询您的数据集][query-nyc]：使用Timescale和PostgreSQL分析包含纽约出租车行程数据的数据集。
3.  [附加：高效存储数据][compress-nyc]：学习如何使用Timescale的压缩功能更高效地存储和查询您的纽约出租车行程数据。

## 关于使用Timescale查询数据

本教程使用[纽约出租车数据][nyc-tlc]向您展示如何构建时间序列数据的查询。您在本教程中进行的分析与数据科学组织用来规划升级、设定预算和分配资源等活动的分析类似。

它首先教您如何设置和连接到Timescale数据库，创建表，并使用`psql`将数据加载到表中。

然后，您将学习如何对您的数据集进行分析和监控。它指导您使用PostgreSQL查询获取信息，包括如何使用JOINs将您的时间序列数据与关系数据或业务数据结合起来。

<PreloadedData />

[dataset-nyc]: /tutorials/:currentVersion:/nyc-taxi-cab/dataset-nyc/
[query-nyc]: /tutorials/:currentVersion:/nyc-taxi-cab/query-nyc/
[compress-nyc]: /tutorials/:currentVersion:/nyc-taxi-cab/compress-nyc/
[advanced-nyc]: /tutorials/:currentVersion:/nyc-taxi-cab/advanced-nyc/
[nyc-tlc]: https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page 
[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account

