---
标题: 摄取实时金融网络套接字数据
摘要: 设置数据管道，以便从不同的金融应用程序编程接口获取数据。
产品: [云服务]
关键词: [金融，分析，网络套接字，数据管道]
标签: [教程，中级]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 摄取实时金融网络套接字数据
---

import CandlestickIntro from "versionContent/_partials/_candlestick_intro.mdx";

# 接入实时金融WebSocket数据

本教程向您展示如何使用WebSocket连接将实时时间序列数据接入TimescaleDB。本教程设置了一个数据管道，从我们的数据中心合作伙伴[Twelve Data][twelve-data]接入实时数据。Twelve Data提供多种不同的金融API，包括股票、加密货币、外汇和ETF。如果您希望频繁更新数据库，它还支持WebSocket连接。使用WebSockets，您需要连接到服务器、订阅符号，您就可以在交易时间内开始接收实时数据。

完成本教程后，您将设置好一个数据管道，将实时金融数据接入您的Timescale。

本教程使用Python和Twelve Data提供的API[包装库][twelve-wrapper]。

## 前提条件

开始之前，请确保您已经：

*   注册了[免费的Timescale账户][cloud-install]。
*   安装了Python 3。
*   注册了[Twelve Data][twelve-signup]。免费层级非常适合本教程。
*   记下您的Twelve Data[API密钥](https://twelvedata.com/account/api-keys)。

## 本教程的步骤

本教程包括：

1.  [设置您的数据集][financial-ingest-dataset]：将数据从[Twelve Data][twelve-data]加载到您的TimescaleDB数据库中。
1.  [查询您的数据集][financial-ingest-query]：创建K线视图，在Grafana中查询聚合数据并可视化数据。

    本教程向您展示如何使用WebSocket连接将实时时间序列数据接入Timescale数据库。创建K线视图，查询聚合数据，并在Grafana中可视化数据。

## 关于OHLCV数据和K线图

<CandlestickIntro />

![candlestick](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/candlestick_fig.png)

TimescaleDB非常适合存储和分析金融K线数据，许多Timescale社区成员正是出于这个目的使用它。

[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account
[financial-ingest-dataset]: /tutorials/:currentVersion:/financial-ingest-real-time/financial-ingest-dataset/
[financial-ingest-query]: /tutorials/:currentVersion:/financial-ingest-real-time/financial-ingest-query/
[twelve-data]: https://twelvedata.com/  
[twelve-signup]: https://twelvedata.com/login  
[twelve-wrapper]: https://github.com/twelvedata/twelvedata-python
