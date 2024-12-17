---
标题: 设计数据库架构
摘要: 设计一个数据库架构来存储你的金融 K 线数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [金融，分析]
标签: [K 线]
---

# 设计数据库模式

设计数据库模式时，您需要考虑它将存储哪些类型的数据。

本教程是关于分析日内股票数据的，因此您需要创建一个能够处理K线数据的模式。一个典型的K线图如下所示：

至少需要四个数据点来创建一个K线图：最高价、开盘价、收盘价和最低价。

![candlestick](https://assets.timescale.com/docs/images/tutorials/intraday-stock-analysis/candlestick_fig.png) 

您还需要拥有股票代码、时间和交易量字段。我们使用的字段如下：

|字段          |描述                      |
|-------------|-------------------------|
|time         |分钟的开始时间            |
|symbol       |股票代码                  |
|price_open   |股票开盘价                |
|price_close  |股票收盘价                |
|price_low    |分钟内的最低价            |
|price_high   |分钟内的最高价            |
|trading_volume |分钟内的交易量          |

基于此，您可以创建一个名为`stocks_intraday`的表：

```sql
CREATE TABLE public.stocks_intraday (
    "time" timestamptz NOT NULL,
    symbol text NULL,
    price_open double precision NULL,
    price_close double precision NULL,
    price_low double precision NULL,
    price_high double precision NULL,
    trading_volume int NULL
);
```

这会创建一个常规的PostgreSQL表，包含所需的所有列以摄取K线数据记录。

# 创建超表

要使用TimescaleDB的功能，您需要启用TimescaleDB，并从`stocks_intraday`表创建一个超表。

**启用TimescaleDB扩展：**

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

**从`stocks_intraday`表创建超表：**

```sql
/*
stocks_intraday: 表名
time: 时间戳列名
*/
SELECT create_hypertable('stocks_intraday', 'time');
```

此时，您拥有一个空的超表，准备摄取时间序列数据。
