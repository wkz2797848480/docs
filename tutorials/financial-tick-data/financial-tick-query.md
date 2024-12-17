---
标题: 分析金融逐笔数据 —— 查询数据
摘要: 创建蜡烛图视图并查询金融逐笔数据以分析价格变化。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，金融，学习]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析金融逐笔数据
---

import GraphOhlcv from "versionContent/_partials/_graphing-ohlcv-data.mdx";

# 查询数据

将原始的实时行情数据转换为聚合的K线视图是处理金融数据的用户的常见任务。TimescaleDB 包括了[超函数][hyperfunctions]，您可以使用它们更轻松地存储和查询您的金融数据。超函数是TimescaleDB中的SQL函数，它们可以让您用更少的代码行在PostgreSQL中操作和分析时间序列数据。

有三个超函数对于计算K线值至关重要：[`time_bucket()`][time-bucket]、[`FIRST()`][first] 和 [`LAST()`][last]。`time_bucket()` 超函数帮助您根据时间戳值将记录聚合到任意时间间隔的桶中。`FIRST()` 和 `LAST()` 帮助您计算开盘价和收盘价。要计算最高价和最低价，您可以使用标准PostgreSQL聚合函数 `MIN` 和 `MAX`。

在TimescaleDB中，创建K线视图的最高效方式是使用[连续聚合][caggs]。
在本教程中，您将为K线时间桶创建一个连续聚合，然后使用不同的刷新策略查询该聚合。最后，您可以使用Grafana将您的数据可视化为K线图。

## 创建连续聚合

要查看OHLCV值，最有效的方式是创建一个连续聚合。在本教程中，您将创建一个连续聚合以聚合每天的数据。然后，您将设置聚合每天刷新，并聚合过去两天的数据。

<Procedure>

### 创建连续聚合

1. 连接到包含Twelve Data加密货币数据集的Timescale数据库。

2. 在psql提示符下，创建连续聚合以每分钟聚合数据：

    ```sql
    CREATE MATERIALIZED VIEW one_day_candle
    WITH (timescaledb.continuous) AS
        SELECT
            time_bucket('1 day', time) AS bucket,
            symbol,
            FIRST(price, time) AS "open",
            MAX(price) AS high,
            MIN(price) AS low,
            LAST(price, time) AS "close",
            LAST(day_volume, time) AS day_volume
        FROM stocks_real_time
        GROUP BY bucket, symbol;
    ```

    当您创建连续聚合时，它默认会刷新。

3. 设置一个刷新策略，如果有过去两天的超表中存在新数据，则每天更新连续聚合：

    ```sql
    SELECT add_continuous_aggregate_policy('one_day_candle',
        start_offset => INTERVAL '3 days',
        end_offset => INTERVAL '1 day',
        schedule_interval => INTERVAL '1 day');
    ```

</Procedure>

## 查询连续聚合

当您设置好连续聚合后，您可以查询它以获取OHLCV值。

<Procedure>

### 查询连续聚合

1. 连接到包含Twelve Data加密货币数据集的Timescale数据库。

2. 在psql提示符下，使用此查询选择过去14天的所有比特币OHLCV数据，按时间桶排序：

    ```sql
    SELECT * FROM one_day_candle
    WHERE symbol = 'BTC/USD' AND bucket >= NOW() - INTERVAL '14 days'
    ORDER BY bucket;
    ```

    查询结果如下所示：

    ```sql
             bucket         | symbol  |  open   |  high   |   low   |  close  | day_volume
    ------------------------+---------+---------+---------+---------+---------+------------
     2022-11-24 00:00:00+00 | BTC/USD |   16587 | 16781.2 | 16463.4 | 16597.4 |      21803
     2022-11-25 00:00:00+00 | BTC/USD | 16597.4 | 16610.1 | 16344.4 | 16503.1 |      20788
     2022-11-26 00:00:00+00 | BTC/USD | 16507.9 | 16685.5 | 16384.5 | 16450.6 |      12300
    ```

</Procedure>

<GraphOhlcv />

[caggs]: /use-timescale/:currentVersion:/continuous-aggregates/
[first]: /api/:currentVersion:/hyperfunctions/first/
[hyperfunctions]: /api/:currentVersion:/hyperfunctions/
[intraday-tutorial]: /tutorials/:currentVersion:/
[last]: /api/:currentVersion:/hyperfunctions/last/
[time-bucket]: /api/:currentVersion:/hyperfunctions/time_bucket/
[lag]: https://www.postgresqltutorial.com/postgresql-lag-function/

