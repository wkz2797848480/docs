---
标题: 分析历史盘中股票数据
摘要: 使用 TimescaleDB 收集、存储并分析盘中股票数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析，psycopg2，Pandas，Plotly]
标签: [K 线]
---

# 分析历史日内股票数据

本教程是一个逐步指南，介绍了如何使用TimescaleDB收集、存储和分析日内股票数据。

本教程有几个主要步骤：

1. 设计数据库模式。
   您将创建一个能够存储1分钟K线数据的表。
2. 获取并摄入股票数据
   您将学习如何从Alpha Vantage API获取数据，并快速将其摄入到数据库中。
3. 探索股票市场数据
   完成所有管道工作后，您可以看到多种方式来探索股票价格点、低点和高点、随时间变化的价格变化、每日涨幅最大的符号、K线图等！

## 前提条件

*   Python 3
*   TimescaleDB（参见[安装选项][install-timescale]）
*   Alpha Vantage API密钥（[免费获取一个][alpha-vantage-apikey]）
*   Virtualenv（安装：`pip install virtualenv`）
*   [Psql][psql-install]或任何其他PostgreSQL客户端（例如，DBeaver）

## 开始：创建虚拟环境

建议创建一个新的Python虚拟环境来隔离本教程中使用的包。

```bash
mkdir intraday-stock-analysis
cd intraday-stock-analysis
virtualenv env
source env/bin/activate
```

在虚拟环境中安装Pandas：

```bash
pip install pandas
```

[alpha-vantage-apikey]: https://www.alphavantage.co/support/#api-key 
[design-schema]: /tutorials/:currentVersion:/
[explore]: /tutorials/:currentVersion:/
[fetch-ingest]: /tutorials/:currentVersion:/
[install-timescale]: /getting-started/latest/
[psql-install]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql
