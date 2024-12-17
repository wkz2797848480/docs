---
标题: 使用 OHLCV（K 线）格式将金融逐笔交易数据存储在 TimescaleDB 中
摘要: 存储你的金融逐笔交易数据，并创建 K 线图视图来分析价格变化。
关键词: [金融，分析]
标签: [K 线]
---

# 使用OHLCV（K线）格式在TimescaleDB中存储金融tick数据

[K线图][charts]是分析金融资产价格变动的标准方式。它们可以用来检查股票价格、加密货币价格甚至NFT价格的趋势。要生成K线图，您需要OHLCV格式的K线数据。也就是说，您需要某些金融资产的开盘价、最高价、最低价、收盘价和成交量数据。

本教程向您展示如何高效地存储原始金融tick数据，创建不同的K线视图，并使用OHLCV格式在TimescaleDB中查询聚合数据。它还向您展示如何下载包含BTC、ETH和其他流行资产的加密货币tick交易的真实世界样本数据。

## 前提条件

在开始之前，请确保您拥有以下条件：

*   本地或云端运行的TimescaleDB实例。更多信息，请参见[入门指南](/getting-started/latest/)
*   [`psql`][psql]、DBeaver或任何其他PostgreSQL客户端

## K线数据和OHLCV是什么？

K线图在金融领域中用于可视化资产价格的变化。每个K线代表一个时间框架（例如，1分钟、5分钟、1小时或类似时间），并显示该时间段内资产价格的变化。

![candlestick](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/candlestick_fig.png) 

K线图是由K线数据生成的，K线数据是图表中使用的数据点集合。这通常缩写为OHLCV（开盘-最高-最低-收盘-成交量）：

*   开盘价：开盘价
*   最高价：最高价
*   最低价：最低价
*   收盘价：收盘价
*   成交量：交易量

这些数据点对应于K线所涵盖的时间桶。例如，1分钟K线将需要那一分钟的开盘价和收盘价。

许多Timescale社区成员使用TimescaleDB来存储和分析K线数据。以下是一些示例：

*   [Trading Strategy如何为加密货币量化交易构建数据栈][trading-strategy]
*   [Messari如何使用数据向所有人开放加密经济][messari]
*   [我如何使用TimescaleDB为（成功的）加密货币交易机器人提供动力][bot]

按照本教程，看看如何设置您的TimescaleDB数据库以高效地消费实时tick或聚合的金融数据，并生成K线视图。

*   [设计模式并摄入tick数据][design]
*   [创建K线（开盘-最高-最低-收盘-成交量）聚合][create]
*   [查询K线视图][query]
*   [高级数据管理][manage]

[charts]: https://www.investopedia.com/terms/c/candlestick.asp 
[trading-strategy]: https://www.timescale.com/blog/how-trading-strategy-built-a-data-stack-for-crypto-quant-trading/ 
[messari]: https://www.timescale.com/blog/how-messari-uses-data-to-open-the-cryptoeconomy-to-everyone/ 
[bot]: https://www.timescale.com/blog/how-i-power-a-successful-crypto-trading-bot-with-timescaledb/ 
[design]: /tutorials/:currentVersion:/financial-candlestick-tick-data/design-tick-schema
[create]: /tutorials/:currentVersion:/financial-candlestick-tick-data/create-candlestick-aggregates
[query]: /tutorials/:currentVersion:/financial-candlestick-tick-data/query-candlestick-views
[manage]: /tutorials/:currentVersion:/financial-candlestick-tick-data/advanced-data-management
[psql]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql/

