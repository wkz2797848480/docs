---
标题: 使用 TimescaleDB 分析金融逐笔数据
摘要: 学习如何存储金融逐笔数据并创建蜡烛图视图以分析价格变化。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，金融，学习]
标签: [教程，初学者]
布局_components: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析金融逐笔数据
---

import CandlestickIntro from "versionContent/_partials/_candlestick_intro.mdx";

# 使用TimescaleDB分析金融行情数据

要分析金融数据，您可以为金融资产绘制开盘、最高、最低、收盘和成交量（OHLCV）信息。利用这些数据，您可以创建K线图，这有助于分析金融资产随时间变化的价格变动。您可以使用K线图检查股票、加密货币或NFT价格的趋势。

在本教程中，您将使用由[Twelve Data][twelve-data]提供的原始金融数据，创建一个聚合的K线视图，查询聚合数据，并在Grafana中可视化数据。

## 前提条件

开始之前，请确保您已具备以下条件：

*   注册了[免费Timescale账户][cloud-install]。

## 本教程的步骤

本教程包括：

1.  [设置您的数据集][financial-tick-dataset]：将数据从[Twelve Data][twelve-data]加载到您的TimescaleDB数据库中。
2.  [查询您的数据集][financial-tick-query]：创建K线视图，查询聚合数据，并在Grafana中可视化数据。
3.  [附加：高效存储数据][financial-tick-compress]：学习如何使用Timescale的压缩功能更高效地存储和查询您的金融行情数据。

    本教程向您展示如何将实时时间序列数据摄入到Timescale数据库中。要创建K线视图，查询聚合数据，并在Grafana中可视化数据，请参阅[实时websocket数据摄入部分][advanced-websocket]。

## 关于OHLCV数据和K线图

<CandlestickIntro />

![candlestick](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/candlestick_fig.png) 

TimescaleDB非常适合存储和分析金融K线数据，许多Timescale社区成员正是出于这个目的使用它。查看一些Timescale社区成员的故事：

*   [Trading Strategy如何为加密货币量化交易构建数据堆栈][trading-strategy]
*   [Messari如何使用数据向所有人开放加密经济][messari]
*   [我如何使用TimescaleDB为（成功的）加密货币交易机器人提供动力][bot]

[advanced-websocket]: /tutorials/:currentVersion:/financial-ingest-real-time/
[cloud-install]: /getting-started/:currentVersion:/#create-your-timescale-account
[financial-tick-dataset]: /tutorials/:currentVersion:/financial-tick-data/financial-tick-dataset/
[financial-tick-query]: /tutorials/:currentVersion:/financial-tick-data/financial-tick-query/
[financial-tick-compress]: /tutorials/:currentVersion:/financial-tick-data/financial-tick-compress/
[twelve-data]: https://twelvedata.com/ 
[trading-strategy]: https://www.timescale.com/blog/how-trading-strategy-built-a-data-stack-for-crypto-quant-trading/ 
[messari]: https://www.timescale.com/blog/how-messari-uses-data-to-open-the-cryptoeconomy-to-everyone/ 
[bot]: https://www.timescale.com/blog/how-i-power-a-successful-crypto-trading-bot-with-timescaledb/

