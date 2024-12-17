---
标题: 创建 K 线聚合数据
摘要: 将原始的金融逐笔交易数据转换为聚合后的 K 线图数据展示形式。
关键词: [金融，分析]
标签: [K 线]
---

# 创建蜡烛图聚合

将原始的实时逐笔数据转换为聚合的蜡烛图视图，是处理金融数据的用户常见的任务。如果你的数据不是逐笔数据，例如你接收到的数据已经是聚合形式，如1分钟桶，你仍然可以使用这些函数帮助你将数据创建成更大的桶，如1小时或1天桶。如果你想使用预聚合的股票和加密货币数据，请参阅[分析日内股票数据][intraday-tutorial]教程以获取更多示例。

TimescaleDB包括[超函数][hyperfunctions]，你可以使用它们来更轻松地存储和查询你的金融数据。超函数是TimescaleDB中的SQL函数，它使得在PostgreSQL中操作和分析时间序列数据更加容易，代码量更少。有三个超函数对于计算蜡烛图值至关重要：[`time_bucket()`][time-bucket]、[`FIRST()`][first]和[`LAST()`][last]。

`time_bucket()`超函数帮助你将记录聚合到任意时间间隔的桶中，基于时间戳值。`FIRST()`和`LAST()`帮助你计算开盘和收盘价。要计算最高和最低价，你可以使用标准的PostgreSQL聚合函数`MIN`和`MAX`。

在这个第一个SQL示例中，使用超函数查询逐笔数据，并将其转换为1分钟蜡烛图值，以蜡烛图格式表示：

```sql
-- 创建蜡烛图格式
SELECT
    time_bucket('1 min', time) AS bucket,
    symbol,
    FIRST(price, time) AS "open",
    MAX(price) AS high,
    MIN(price) AS low,
    LAST(price, time) AS "close",
    LAST(day_volume, time) AS day_volume
FROM crypto_ticks
GROUP BY bucket, symbol
```

此查询中的超函数：

*   `time_bucket('1 min', time)`：创建1分钟桶
*   `FIRST(price, time)`：选择桶中的第一个`price`值，按`time`排序，即蜡烛图的开盘价。
*   `LAST(price, time)`选择桶中的最后一个`price`值，按`time`排序，即蜡烛图的收盘价

除了超函数，你还可以看到这里有其他常见的SQL聚合函数，如`MIN`和`MAX`，它们计算蜡烛图中的最低和最高价格。

<Highlight type="note">
这个教程使用`LAST()`超函数来计算桶内的交易量，因为示例逐笔数据已经提供了增量的`day_volume`字段，该字段包含了给定天的每次交易的总交易量。根据你接收到的原始数据以及你是否想以交易计数或交易总价值来计算交易量，你可能需要使用`COUNT(*)`、`SUM(price)`或桶中最后和第一个值的差值来获得正确的结果。
</Highlight>

## 创建蜡烛图数据的连续聚合

在TimescaleDB中，创建蜡烛图视图的最高效方式是使用[连续聚合][caggs]。连续聚合与PostgreSQL物化视图非常相似，但有三个主要优势。

首先，
物化视图在每次刷新视图时会重新创建所有数据，这会导致历史数据丢失。连续聚合只刷新源生数据已更改或添加的聚合数据桶。

其次，连续聚合可以使用内置的、用户配置的政策自动刷新。不需要特殊的触发器或存储过程来随时间刷新数据。

最后，连续聚合默认是实时的。任何在刷新之间插入的新原始逐笔数据都会自动追加到物化数据中。这保持了你的蜡烛图数据的最新状态，而无需编写特殊的SQL来从多个视图和表联合数据。

连续聚合通常用于支持仪表板和其他面向用户的应用程序，如价格图表，其中查询性能和数据的及时性很重要。

让我们看看如何使用不同的刷新策略创建不同的蜡烛图时间桶 - 1分钟、1小时和1天 - 使用连续聚合。

### 1分钟蜡烛图

要创建1分钟蜡烛图的连续聚合，使用你之前用来获取1分钟OHLCV值的相同查询。但这次，将查询放入连续聚合定义中：

```sql
/* 1分钟蜡烛图视图*/
CREATE MATERIALIZED VIEW one_min_candle
WITH (timescaledb.continuous) AS
    SELECT
        time_bucket('1 min', time) AS bucket,
        symbol,
        FIRST(price, time) AS "open",
        MAX(price) AS high,
        MIN(price) AS low,
        LAST(price, time) AS "close",
        LAST(day_volume, time) AS day_volume
    FROM crypto_ticks
    GROUP BY bucket, symbol
```

当你运行这个查询时，TimescaleDB查询所有你的逐笔数据的1分钟聚合值，创建连续聚合并物化结果。但你的蜡烛图数据只物化到了最后一个数据点。如果你想让连续聚合随着新数据的不断到来而保持最新，你还需要添加一个连续聚合刷新策略。例如，每两分钟刷新一次连续聚合：

```sql
/* 每两分钟刷新一次连续聚合 */
SELECT add_continuous_aggregate_policy('one_min_candle',
    start_offset => INTERVAL '2 hour',
    end_offset => INTERVAL '10 sec',
    schedule_interval => INTERVAL '2 min');
```

连续聚合每小时刷新一次，所以每小时都会物化新的蜡烛图，**如果超表中有新的原始逐笔数据**。

当这个作业运行时，它只刷新`start_offset`和`end_offset`之间的时间段，并忽略此窗口外的修改。

在大多数情况下，将`end_offset`设置为与连续聚合定义中的时间桶相同或更大。这确保只有在刷新过程中物化完整的桶。

### 1小时蜡烛图

要创建1小时蜡烛图视图，按照前一步的相同过程操作，只是这次在连续聚合定义中将时间桶值设置为一小时：

```sql
/* 1小时蜡烛图视图 */
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
    FROM crypto_ticks
    GROUP BY bucket, symbol
```

添加一个刷新策略，每小时刷新一次连续聚合：

```sql
/* 每小时刷新一次连续聚合 */
SELECT add_continuous_aggregate_policy('one_hour_candle',
    start_offset => INTERVAL '1 day',
    end_offset => INTERVAL '1 min',
    schedule_interval => INTERVAL '1 hour');
```

注意这个示例使用了不同的刷新策略和不同的参数值来适应连续聚合定义中的1小时时间桶。连续聚合将每小时刷新一次，所以每小时都会有新的蜡烛图数据物化，如果超表中有新的原始逐笔数据。

### 1天蜡烛图

使用上述相同的过程创建本教程的最终视图，用于1天蜡烛图，使用1天的时间桶大小：

```sql
/* 1天蜡烛图 */
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
    FROM crypto_ticks
    GROUP BY bucket, symbol
```

添加一个刷新策略，每天刷新一次连续聚合：

```sql
/* 每天刷新一次连续聚合 */
SELECT add_continuous_aggregate_policy('one_day_candle',
    start_offset => INTERVAL '3 day',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

刷新作业每天运行一次，并物化两天的蜡烛图。

## 可选：在蜡烛图视图中添加价格变化（delta）列

作为可选步骤，你可以在连续聚合中添加一个额外的列来计算桶内开盘价和收盘价之间的价格差异。

通常，你可以使用以下公式计算价格差异：

```text
(收盘价 - 开盘价) / 开盘价 = delta
```

在SQL中计算delta：

```sql
SELECT time_bucket('1 day', time) AS bucket, symbol, (LAST(price, time)-FIRST(price, time))/FIRST(price, time) AS change_pct
FROM crypto_ticks
WHERE price != 0
GROUP BY bucket, symbol
```

带有价格变化列的1天蜡烛图的完整连续聚合定义：

```sql
/* 带有价格变化列的1天蜡烛图*/
CREATE MATERIALIZED VIEW one_day_candle_delta
WITH (timescaledb.continuous) AS
    SELECT
        time_bucket('1 day', time) AS bucket,
        symbol,
        FIRST(price, time) AS "open",
        MAX(price) AS high,
        MIN(price) AS low,
        LAST(price, time) AS "close",
        LAST(day_volume, time) AS day_volume,
        (LAST(price, time)-FIRST(price, time))/FIRST(price, time) AS change_pct
    FROM crypto_ticks
    WHERE price != 0
    GROUP BY bucket, symbol
```

## 使用多个连续聚合

你目前不能在另一个连续聚合上创建连续聚合。
然而，在大多数情况下，这不是必要的。你可以通过为同一超表
创建多个连续聚合来获得类似的结果和性能。由于连续聚合的高效物化机制，刷新和查询性能都应该表现良好。

[caggs]: /use-timescale/:currentVersion:/continuous-aggregates/
[first]: /api/:currentVersion:/hyperfunctions/first/
[hyperfunctions]: /api/:currentVersion:/hyperfunctions/
[intraday-tutorial]: /tutorials/:currentVersion:/
[last]: /api/:currentVersion:/hyperfunctions/last/
[time-bucket]: /api/:currentVersion:/hyperfunctions/time_bucket/

