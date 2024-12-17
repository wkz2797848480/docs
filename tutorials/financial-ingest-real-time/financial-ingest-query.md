---
标题: 摄取实时金融网络套接字数据-查询数据
摘要: 创建蜡烛图视图并查询金融逐笔数据以分析价格变化。
产品: [云服务]
关键词: [金融，分析，网络套接字，数据管道]
标签: [教程，中级]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 摄取实时金融网络套接字数据
---

import GraphOhlcv from "versionContent/_partials/_graphing-ohlcv-data.mdx";

# 查询数据

要查看OHLCV值，最有效的方法是创建一个连续聚合。您可以创建一个连续聚合来每小时聚合数据，然后设置聚合每小时刷新，并聚合最后两小时的数据。

<Procedure>

## 创建连续聚合

1.  连接到包含十二数据股票数据集的Timescale数据库`tsdb`。

2.  在psql提示符下，创建连续聚合以每分钟聚合数据：

    ```sql
    CREATE MATERIALIZED VIEW one_hour_candle
    WITH (timescaledb.continuous) AS
        SELECT
            time_bucket('1 hour', time) AS bucket,
            symbol,
            FIRST(price, time) AS "open",
            MAX(price) AS high,
            MIN(price) AS low,
            LAST(price, time) AS "close",
            LAST(day_volume, time) AS day_volume
        FROM stocks_real_time
        GROUP BY bucket, symbol;
    ```

    创建连续聚合时，默认会刷新。

3.  设置刷新策略，如果超表中最后两小时有新数据可用，则每小时更新连续聚合：

    ```sql
    SELECT add_continuous_aggregate_policy('one_hour_candle',
        start_offset => INTERVAL '3 hours',
        end_offset => INTERVAL '1 hour',
        schedule_interval => INTERVAL '1 hour');
    ```

</Procedure>

## 查询连续聚合

设置好连续聚合后，您可以查询它以获取OHLCV值。

<Procedure>

### 查询连续聚合

1.  连接到包含十二数据股票数据集的Timescale数据库。

2.  在psql提示符下，使用此查询按时间桶选择过去5小时所有`AAPL`的OHLCV数据：

    ```sql
    SELECT * FROM one_hour_candle
    WHERE symbol = 'AAPL' AND bucket >= NOW() - INTERVAL '5 hours'
    ORDER BY bucket;
    ```

    查询结果如下所示：

    ```sql
             bucket         | symbol  |  open   |  high   |   low   |  close  | day_volume
    ------------------------+---------+---------+---------+---------+---------+------------
     2023-05-30 08:00:00+00 | AAPL   | 176.31 | 176.31 |    176 | 176.01 |
     2023-05-30 08:01:00+00 | AAPL   | 176.27 | 176.27 | 176.02 |  176.2 |
     2023-05-30 08:06:00+00 | AAPL   | 176.03 | 176.04 | 175.95 |    176 |
     2023-05-30 08:07:00+00 | AAPL   | 175.95 |    176 | 175.82 | 175.91 |
     2023-05-30 08:08:00+00 | AAPL   | 175.92 | 176.02 |  175.8 | 176.02 |
     2023-05-30 08:09:00+00 | AAPL   | 176.02 | 176.02 |  175.9 | 175.98 |
     2023-05-30 08:10:00+00 | AAPL   | 175.98 | 175.98 | 175.94 | 175.94 |
     2023-05-30 08:11:00+00 | AAPL   | 175.94 | 175.94 | 175.91 | 175.91 |
     2023-05-30 08:12:00+00 | AAPL   |  175.9 | 175.94 |  175.9 | 175.94 |
    ```

</Procedure>

<GraphOhlcv />

