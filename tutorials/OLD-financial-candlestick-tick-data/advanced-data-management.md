---
标题: 高级数据管理
摘要: 学习用于长期管理你的逐笔交易数据和 K 线数据的高级技巧。
关键词: [金融，分析]
标签: [K 线]
---

# 高级数据管理

本教程的最后部分向您展示了一些更高级的技术，以高效地长期管理您的tick数据和K线数据。TimescaleDB配备了多项功能，帮助您管理数据生命周期，并随着数据增长减少您的磁盘存储需求。

本节包含四个示例，展示如何在您的tick数据 hypertable 和 K线连续聚合上设置自动化策略。这可以帮助您通过自动执行以下操作来节省磁盘存储空间并提高远程分析查询的性能：
<!-- vale Google.LyHyphens = NO -->
*   [删除旧的tick数据](#自动删除旧的tick数据)
*   [删除旧的K线数据](#自动删除旧的K线数据)
*   [压缩tick数据](#自动压缩tick数据)
*   [压缩K线数据](#自动压缩K线数据)
<!-- vale Google.LyHyphens = YES -->

在实施这些自动化策略之前，重要的是要对TimescaleDB hypertables 和连续聚合中的块时间间隔有一个高层次的理解。您为tick数据表设置的块时间间隔直接影响这些自动化策略的工作方式。更多信息，请参见[hypertables和chunks][chunks]部分。

## Hypertable块时间间隔和自动化策略

TimescaleDB使用hypertables提供高层次且熟悉的抽象层，与PostgreSQL表进行交互。您只需要访问一个hypertable就可以访问您的所有时间序列数据。

在底层，TimescaleDB根据时间戳列创建块。每个块的大小由[`chunk_time_interval`][interval]参数决定。您可以在创建hypertable时提供此参数，也可以之后更改它。如果您不提供此可选参数，块时间间隔默认为7天。这意味着hypertable中的每个块包含7天的数据。

了解您的块时间间隔非常重要。本节描述的所有TimescaleDB自动化策略都依赖于此信息，块时间间隔从根本上影响这些策略对您的数据的影响。

在本节中，了解这些自动化策略以及它们在金融tick数据的上下文中如何工作。

## 自动删除旧的tick数据

通常，您的时间序列数据越旧，其相关性和实用性就越低。这通常也适用于tick数据。随着时间的推移，您可能不再需要原始tick数据，因为您只想查询K线聚合。在这种情况下，您可以决定在tick数据超过一定时间间隔后自动从hypertable中删除。

TimescaleDB有内置的方法在特定时间后自动删除原始数据。您可以使用[data retention policy][retention]设置此自动化：

```sql
SELECT add_retention_policy('crypto_ticks', INTERVAL '7 days');
```

当您运行此操作时，它会向`crypto_ticks` hypertable添加一个数据保留策略，该策略在块中的所有数据超过7天后删除该块。块中的所有记录都需要超过7天才能删除该块。

了解您的hypertable的块时间间隔至关重要。如果您设置了一个`INTERVAL '3 days'`的数据保留策略，该策略在三天后不会删除任何数据，因为您的块时间间隔是七天。即使已经过去了三天，最近的块仍然包含超过三天的新数据，因此不能被数据保留策略删除。

如果您想改变这种行为，并更频繁、更早地删除块，请尝试使用不同的块时间间隔。例如，如果您将块时间间隔设置为仅两天，您可以创建一个2天间隔的保留策略，每两天删除一个块（假设您在此期间正在摄取数据）。

更多信息，请参见[data retention][retention]部分。

<Highlight type="important">
确保没有任何连续聚合策略与数据保留策略相交。只有先将数据在连续聚合中物化，然后才能从底层hypertable中删除数据，才能保留连续聚合中的K线数据并从底层hypertable中删除tick数据。
</Highlight>

## 自动删除旧的K线数据

从您的hypertable中删除旧的原始tick数据，同时保留聚合视图的时间更长，是最小化磁盘使用的一种常见方式。然而，从连续聚合中删除旧的K线数据可以提供另一种方法，进一步控制长期磁盘使用。
TimescaleDB也允许您在连续聚合上创建数据保留策略。

<Highlight type="note">
连续聚合也有块时间间隔，因为它们在后台使用hypertables。默认情况下，连续聚合的块时间间隔是原始hypertable的块时间间隔的10倍。例如，如果原始hypertable的块时间间隔是7天，那么上面的连续聚合将有70天的块时间间隔。
</Highlight>

您可以设置数据保留策略以从`one_min_candle`连续聚合中删除旧数据：

```sql
SELECT add_retention_policy('one_min_candle', INTERVAL '70 days');
```

此数据保留策略删除连续聚合中超过70天的块。在TimescaleDB中，这是由hypertable的`range_end`属性决定的，或者在连续聚合的情况下，是物化的hypertable。实际上，这意味着如果您为具有70天`chunk_time_interval`的连续聚合定义了30天的数据保留策略，数据将不会从连续聚合中删除，直到块的`range_end`至少比当前时间早70天，这是由于原始hypertable的块时间间隔。

## 自动压缩tick数据

TimescaleDB允许您保留hypertable中的tick数据，但仍然可以通过TimescaleDB的本地压缩节省存储成本。您需要在hypertable上启用压缩并设置压缩策略以自动压缩旧数据。

在`crypto_ticks` hypertable上启用压缩：

```sql
ALTER TABLE crypto_ticks SET (
 timescaledb.compress,
 timescaledb.compress_segmentby = 'symbol'
);
```

设置压缩策略以压缩超过7天的数据：

```sql
SELECT add_compression_policy('crypto_ticks', INTERVAL '7 days');
```

执行这两个SQL脚本将压缩超过7天的块。

更多信息，请参见[compression][compression]部分。

## 自动压缩K线数据

从[TimescaleDB 2.6][release-blog]开始，您也可以在您的连续聚合上设置压缩策略。如果您存储了大量的历史K线数据，这将是一个有用的功能，这些数据消耗了大量的磁盘空间，但您仍然希望长期保留它们。

在`one_min_candle`视图上启用压缩：

```sql
ALTER MATERIALIZED VIEW one_min_candle set (timescaledb.compress = true);
```

添加压缩策略以在70天后压缩数据：

```sql
SELECT add_compression_policy('one_min_candle', compress_after=> INTERVAL '70 days');
```

<Highlight type="important">
在设置任何K线视图的压缩策略之前，请先设置刷新策略。压缩策略间隔应设置为不压缩积极刷新的时间间隔。
</Highlight>

[阅读更多关于压缩连续聚合的信息。][caggs-compress]

[caggs-compress]: /use-timescale/:currentVersion:/continuous-aggregates/compression-on-continuous-aggregates/
[chunks]: /use-timescale/:currentVersion:/hypertables/about-hypertables/
[compression]: /use-timescale/:currentVersion:/compression/
[interval]: /api/:currentVersion:/hypertable/set_chunk_time_interval/
[release-blog]: https://www.timescale.com/blog/increase-your-storage-savings-with-timescaledb-2-6-introducing-compression-for-continuous-aggregates/ 
[retention]: /use-timescale/:currentVersion:/data-retention/create-a-retention-policy/

