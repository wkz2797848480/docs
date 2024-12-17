---
标题: 使用连续聚合和超级函数分析数据
摘要: 学习如何利用 TimescaleDB 的功能高效地分析时间序列数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [连续聚合，超级函数，分析]
---

# 使用TimescaleDB连续聚合和超函数分析数据

本教程是关于如何使用TimescaleDB分析时间序列数据的逐步指南。我们将向您展示如何利用TimescaleDB的连续聚合和超函数进行更快速、更高效的查询。我们还利用了TimescaleDB的独特能力：将时间序列数据与关系数据进行连接。

我们使用的数据集由国家橄榄球联盟（NFL）提供，包含2018年NFL赛季所有传球比赛的球员和追踪数据。我们将使用Python将这个数据集导入TimescaleDB，并开始探索它，以发现有关球员和球队的洞见。

如果您恰好是NFL梦幻橄榄球玩家，使用这些对过去数据的分析可能有助于您为即将到来的年份选择最有效的球员。而且，随着NFL在整个赛季中发布新数据，您可以导入这些数据，帮助您从一周到另一周做出更好的决策。

即使您不是NFL粉丝，本教程也提供了一个很好的示例，展示了如何将时间序列数据导入TimescaleDB（即使它看起来不像时间序列数据），如何使用普通的SQL和TimescaleDB超函数进行强大的数据分析，以及如何使用Python进行数据可视化。

本教程有几个部分可以帮助您在您的旅程中：

1. 导入和查询数据
   下载数据，在TimescaleDB中创建表，并运行您对NFL追踪数据的第一个查询。
2. 使用连续聚合和超函数分析数据
   使用更高级的查询深入检查数据，利用TimescaleDB的功能使查询更快、更有效。您还将看到一些使用数据创建的可视化示例。
3. 将时间序列数据与关系数据连接
   通过将时间序列数据与关系数据连接，进一步了解您的时间序列数据。
4. 可视化时间序列逐场数据
   为了增加一些额外的乐趣，使用Python和Matplotlib创建图像，绘制任何比赛中场上每个球员的移动轨迹。

## 前提条件

*   Python 3
*   TimescaleDB（参见[安装选项][install-timescale]）
*   [Psql][psql-install]或任何其他PostgreSQL客户端（例如，DBeaver）
*   [Timescale工具包][toolkit]

## 下载数据集

*   [NFL数据集可在Kaggle上下载。][kaggle-download]
*   [额外的体育场和得分数据集（.zip）（来源：wikipedia.com）。][extra-download]

## 资源

*   [Kaggle上的NFL Big Data Bowl 2021](https://www.kaggle.com/c/nfl-big-data-bowl-2021) 

[analyze-data]: /tutorials/:currentVersion:/nfl-analytics/advanced-analysis/
[extra-download]: https://assets.timescale.com/docs/downloads/nfl_2018.zip 
[ingest-query]: /tutorials/:currentVersion:/nfl-analytics/ingest-and-query
[install-timescale]: /getting-started/latest/
[join-data]: /tutorials/:currentVersion:/nfl-analytics/join-with-relational
[kaggle-download]: https://www.kaggle.com/c/nfl-big-data-bowl-2021/data 
[psql-install]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql
[toolkit]: /self-hosted/:currentVersion:/tooling/install-toolkit/
[visualize-plays]: /tutorials/:currentVersion:/nfl-analytics/play-visualization/

