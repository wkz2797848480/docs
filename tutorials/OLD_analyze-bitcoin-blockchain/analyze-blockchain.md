---
标题: 使用超函数分析区块链
摘要: 运用 SQL 函数和超函数来分析比特币交易。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一篇 / 下一篇按钮]
内容分组: 分析比特币区块链
---

# 使用超函数分析区块链

在本节中，我们将使用SQL以不同方式分析比特币交易。看看超函数和连续聚合如何使查询和分析区块链数据变得更容易。

## 简化统计查询的超函数

在以下一些查询中，您会发现一些自定义的SQL函数，它们不是标准PostgreSQL的一部分。这些查询是TimescaleDB的[超函数][docs-hyperfunctions]，它们是TimescaleDB扩展或工具包扩展的一部分。超函数是一系列SQL函数，使在PostgreSQL中操作和分析时间序列数据变得更容易。您需要[安装并启用工具包扩展][install-toolkit]才能使用全套超函数并成功运行以下查询。

安装扩展后，启用它：

```sql
CREATE EXTENSION timescaledb_toolkit;
```

现在，设置一些连续聚合以进行更快和简化的分析。

## 区块链分析的连续聚合

[连续聚合][docs-cagg]是时间序列数据的物质化视图。它们通过持续物化聚合数据使查询更快。同时，它们提供实时结果。这意味着它们包括底层超表中的最新数据。

通过使用连续聚合，您简化并加快了查询速度。

在本教程中，您将创建三个连续聚合，重点关注数据集的三个方面：

*   比特币交易
*   比特币区块
*    Coinbase交易（矿工收入）

预聚合您的数据很重要，因为数据集包含大量交易：每小时超过10,000笔。

在本节中，您还将学习如何通过自动刷新策略使您的连续聚合视图保持最新。

### 连续聚合：交易

创建一个名为`one_hour_transactions`的连续聚合。此视图保存每小时交易的聚合数据。

```sql
CREATE MATERIALIZED VIEW one_hour_transactions
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
   count(*) AS tx_count,
   sum(fee) AS total_fee_sat,
   sum(fee_usd) AS total_fee_usd,
   stats_agg(fee) AS stats_fee_sat,
   avg(size) AS avg_tx_size,
   avg(weight) AS avg_tx_weight,
   count(
         CASE
            WHEN (fee > output_total) THEN hash
            ELSE NULL
         END) AS high_fee_count
  FROM transactions
  WHERE (is_coinbase IS NOT TRUE)
GROUP BY bucket;
```

在此查询中，您在连续聚合中创建了这些聚合列：

*   `tx_count`：总交易量
*   `total_fee_sat`：支付的总费用（以Sat为单位）
*   `total_fee_usd`：支付的总费用（以USD为单位）
*   `stats_fee_sat`：费用统计（以Sat为单位）
    此列使用了一个名为[`stats_agg`][stats_agg]的超函数。
    原始的`stats_agg`值不易解释。
    稍后，您可以使用`stats_agg`计算其他统计数据，例如平均值。
*   `avg_tx_size`：平均交易大小（以KB为单位）
*   `high_fee_count`：费用高于交易量的交易数量

添加刷新策略以保持连续聚合最新：

```sql
SELECT add_continuous_aggregate_policy('one_hour_transactions',
   start_offset => INTERVAL '3 hours',
   end_offset => INTERVAL '1 hour',
   schedule_interval => INTERVAL '1 hour');
```

### 连续聚合：区块

创建一个名为`one_hour_blocks`的连续聚合。此视图保存每小时开采的所有区块的聚合数据。

```sql
CREATE MATERIALIZED VIEW one_hour_blocks
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
   block_id,
   count(*) AS tx_count,
   sum(fee) AS block_fee_sat,
   sum(fee_usd) AS block_fee_usd,
   stats_agg(fee) AS stats_tx_fee_sat,
   avg(size) AS avg_tx_size,
   avg(weight) AS avg_tx_weight,
   sum(size) AS block_size,
   sum(weight) AS block_weight,
   max(size) AS max_tx_size,
   max(weight) AS max_tx_weight,
   min(size) AS min_tx_size,
   min(weight) AS min_tx_weight
FROM transactions
WHERE is_coinbase IS NOT TRUE
GROUP BY bucket, block_id;
```

运行此查询时，您在连续聚合中创建了这些聚合列：

*   `tx_count`：每个区块的交易数量
*   `block_fee_sat`：每个区块支付的交易费用（以Sat为单位）
*   `block_fee_usd`：每个区块支付的交易费用（以USD为单位）
*   `stats_tx_fee_sat`：交易费用统计
*   `avg_tx_size`：区块内的平均交易大小
*   `avg_tx_weight`：区块内的平均交易权重
*   `block_size`：总区块大小
*   `block_weight`：总区块权重
*   `max_tx_size`：区块内的最大交易大小
*   `max_tx_weight`：区块内的最大交易权重
*   `min_tx_size`：区块内的最小交易大小
*   `min_tx_weight`：区块内的最小交易权重

添加刷新策略以保持连续聚合最新：

```sql
SELECT add_continuous_aggregate_policy('one_hour_blocks',
   start_offset => INTERVAL '3 hours',
   end_offset => INTERVAL '1 hour',
   schedule_interval => INTERVAL '1 hour');
```

## 连续聚合：Coinbase交易（矿工收入）

创建一个名为`one_hour_coinbase`的连续聚合。此视图保存每小时矿工作为奖励收到的所有交易的聚合数据。

```sql
CREATE MATERIALIZED VIEW one_hour_coinbase
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS bucket,
   count(*) AS tx_count,
   stats_agg(output_total, output_total_usd) AS stats_miner_revenue,
   min(output_total) AS min_miner_revenue,
   max(output_total) AS max_miner_revenue
FROM transactions
WHERE is_coinbase IS TRUE
GROUP BY bucket;
```

运行此查询时，您在连续聚合中创建了这些聚合列：

*   `tx_count`：每天的Coinbase交易数量
*   `stats_miner_revenue`：矿工收入统计（包括Sat和USD）
*   `min_miner_revenue`：每天挖掘一个区块后矿工收入的最低值
*   `max_miner_revenue`：每天挖掘一个区块后矿工收入的最高值

添加刷新策略以保持连续聚合最新：

```sql
SELECT add_continuous_aggregate_policy('one_hour_coinbase',
   start_offset => INTERVAL '3 hours',
   end_offset => INTERVAL '1 hour',
   schedule_interval => INTERVAL '1 hour');
```

在每个连续聚合定义中，`time_bucket()`函数控制时间桶的大小。所有示例都使用1小时的时间桶。

## 使用SQL生成洞察

以下是您可能对区块链交易、区块和矿工收入提出的一些问题。对于每个问题，您都会得到一个相关的SQL查询和一个回答问题的图表。

**问题**

*   [交易数量和交易费用之间有任何联系吗？](#交易数量和交易费用之间有任何联系吗)
*   [交易量是否影响BTC-USD汇率？](#交易量是否影响BTC-USD汇率)
*   [区块中的交易越多，区块的开采成本就越高吗？](#区块中的交易越多，区块的开采成本就越高吗)
*   [平均矿工收入中有多少百分比来自费用与区块奖励？](#平均矿工收入中有多少百分比来自费用与区块奖励)
*   [区块权重如何影响矿工费用？](#区块权重如何影响矿工费用)
*   [每个区块的平均矿工收入是多少？](#每个区块的平均矿工收入是多少)



### 交易数量与交易费用之间是否存在关联？

交易费用是区块链用户非常关注的问题。如果一个区块链网络的费用过高，你可能就不想使用它。这个查询展示了比特币交易数量与费用之间是否存在相关性。这次分析的时间范围是最近一天。

<Terminal>

<tab label='SQL'>

```sql
SELECT
 bucket AS "time",
 tx_count as "tx volume",
 average(stats_fee_sat) as fees
FROM one_hour_transactions
WHERE bucket > NOW() - INTERVAL '1 day'
ORDER BY 1
```

</tab>

<tab label="数据">

```
time               |tx volume|fees              |
2022-05-21 08:00:00|     5949| 6251.997814758783|
2022-05-21 09:00:00|     5558| 5036.266642677222|
2022-05-21 10:00:00|     1411|13300.024096385541|
2022-05-21 11:00:00|     8488|7209.5062441093305|
2022-05-21 12:00:00|    15741| 4380.564830696906|
```

</tab>

</Terminal>

![最近一天每小时交易量和费用的图表](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/tx_volume_fees.png) 

在这张图表中，绿线表示随时间变化的平均交易量。黄线表示随时间变化的每笔交易的平均费用。这些趋势可能帮助你决定是现在提交交易，还是等几天费用降低后再提交。

### 交易量是否影响BTC-USD汇率？

在加密货币交易中，有很多猜测。你可以通过查看区块链指标之间的相关性，如交易量和费用，来采取基于数据的交易策略。

<Terminal>

<tab label='SQL'>

```sql
SELECT
 bucket AS "time",
 tx_count as "tx volume",
 total_fee_usd / (total_fee_sat*0.00000001) AS "btc-usd rate"
FROM one_hour_transactions
WHERE bucket > NOW() - INTERVAL '1 day'
ORDER BY 1
```

</tab>

<tab label="数据">

```
time               |tx volume|btc-usd rate      |
2022-05-21 08:00:00|     5949|29292.908489698697|
2022-05-21 09:00:00|     5558|29292.881035254628|
2022-05-21 10:00:00|     1411|29292.947146736336|
2022-05-21 11:00:00|     8488|29292.943496737145|
2022-05-21 12:00:00|    15741| 29292.91300051999|
```

</tab>

</Terminal>

![最近一天每小时交易量和BTC-USD转换率的图表](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/volume_btc_usd.png) 

再次，绿线显示了随时间变化的平均交易量。黄线显示了BTC-USD转换率。

接下来，通过分析交易和区块之间的联系，获得区块级别的洞察。

### 区块中更多的交易是否意味着区块更昂贵？

看看区块中交易数量如何影响整体区块挖矿费用。对于这次分析，你可能想要查看更大的时间范围。将分析的时间范围更改为最近五天。

<Terminal>

<tab label='SQL'>

```sql
SELECT
 bucket as "time",
 avg(tx_count) AS transactions,
 avg(block_fee_sat)*0.00000001 AS "mining fee"
FROM one_hour_blocks
WHERE bucket > now() - INTERVAL '5 day'
GROUP BY bucket
ORDER BY 1
```

</tab>

<tab label="数据">

```
time               |transactions         |mining fee |
2022-05-21 08:00:00|1189.8000000000000000|0.07438627000000000000|
2022-05-21 09:00:00| 926.3333333333333333|0.04665261666666666667|
2022-05-21 10:00:00|1411.0000000000000000|0.18766334000000000000|
2022-05-21 11:00:00|2829.3333333333333333|0.20398096333333333333|
2022-05-21 12:00:00|1749.0000000000000000|0.07661607888888888889|
```

</tab>

</Terminal>

![最近五天区块平均交易数量和区块挖矿费用的折线图](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/tx_in_block_expensive.png) 

毫不奇怪，区块中的交易数量和挖矿费用之间存在很高的相关性。区块中的交易越多，区块挖矿费用就越高。

在下一个查询中，看看区块重量和挖矿费用之间是否存在相同的相关性。（区块重量是区块大小的度量单位）。更多的交易应该会增加区块重量，从而提高矿工费用。这个查询与前一个查询非常相似：

<Terminal>

<tab label='SQL'>

```sql
SELECT
 bucket as "time",
 avg(block_weight) as "block weight",
 avg(block_fee_sat*0.00000001) as "mining fee"
FROM one_hour_blocks
WHERE bucket > now() - INTERVAL '5 day'
group by bucket
ORDER BY 1
```

</tab>

<tab label="数据">

```
time               |block weight        |mining fee            |
2022-05-21 08:00:00|2465442.600000000000|0.07438627000000000000|
2022-05-21 09:00:00|1559434.000000000000|0.04665261666666666667|
2022-05-21 10:00:00|3991686.000000000000|0.18766334000000000000|
2022-05-21 11:00:00|3993417.000000000000|0.20398096333333333333|
2022-05-21 12:00:00|3171876.222222222222|0.07661607888888888889|
```

</tab>

</Terminal>

![最近五天区块重量和挖矿费用的折线图](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/weight_fee.png) 

你可以看到区块重量（以重量单位定义）和挖矿费用之间存在相同类型的高相关性。当区块重量接近其最大值（400万重量单位）时，这种关系会减弱，在这种情况下，区块无法包含更多的交易。

在之前的图表中，你看到了挖矿费用与区块重量和交易量之间的相关性。在下一个查询中，从不同的角度分析数据。矿工收入不仅由矿工费用组成，还包括挖出新区块后的区块奖励。目前的奖励是6.25 BTC，并且每四年减半一次。矿工收入有哪些趋势？

### 平均矿工收入中有多少百分比来自费用与区块奖励相比？

矿工因为挖矿每个区块而获得费用和奖励，从而激励他们保持网络的运行。他们的收入有多少来自每个来源？

<Terminal>

<tab label='SQL'>

```sql
WITH coinbase AS (
   SELECT block_id, output_total AS coinbase_tx FROM transactions
   WHERE is_coinbase IS TRUE and time > NOW() - INTERVAL '5 days'
)
SELECT
   bucket as "time",
   avg(block_fee_sat)*0.00000001 AS "fees",
   FIRST((c.coinbase_tx - block_fee_sat), bucket)*0.00000001 AS "reward"
FROM one_hour_blocks b
INNER JOIN coinbase c ON c.block_id = b.block_id
GROUP BY bucket
ORDER BY 1;
```

</tab>

<tab label="数据">

```
time               |fees                  |reward    |
2022-05-21 08:00:00|0.07438627000000000000|6.25000000|
2022-05-21 09:00:00|0.04665261666666666667|6.25000000|
2022-05-21 10:00:00|0.18766334000000000000|6.25000000|
2022-05-21 11:00:00|0.20398096333333333333|6.25000000|
2022-05-21 12:00:00|0.07661607888
888888889|6.25000000|
```

</tab>

</Terminal>

![最近五天平均矿工收入的折线图](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/revenue_ratio.png) 

这张图表分析了最近五天的平均矿工收入。左侧的轴显示了总收入中来自交易费用（绿色）和区块奖励（黄色）的百分比。实际上，大多数矿工收入来自区块奖励（目前是6.25 BTC）。在过去五天中，费用从未超过总收入的3%。

这种分析可以引发关于区块奖励长期减少以及链上费用需要上升以激励矿工和维持网络的讨论。（请注意，左侧的轴是对数刻度，这样可以更容易地看到绿色“费用”部分。）

### 区块重量如何影响矿工费用？

你已经看到区块中更多的交易意味着它更昂贵。区块重量呢？区块中的交易越多，它的大小（或重量）就越大，所以区块重量和挖矿费用应该是紧密相关的。

这个查询使用12小时移动平均值来计算随时间变化的区块重量和区块挖矿费用。

<Terminal>

<tab label='SQL'>

```sql
WITH stats AS (
   SELECT
       bucket,
       stats_agg(block_weight, block_fee_sat) AS block_stats
   FROM one_hour_blocks
   WHERE bucket > NOW() - INTERVAL '5 days'
   GROUP BY bucket
)
SELECT
   bucket as "time",
   average_y(rolling(block_stats) OVER (ORDER BY bucket RANGE '12 hours' PRECEDING)) AS "block weight",
   average_x(rolling(block_stats) OVER (ORDER BY bucket RANGE '12 hours' PRECEDING))*0.00000001 AS "mining fee"
FROM stats
ORDER BY 1
```

</tab>

<tab label="数据">

```
time               |block weight      |mining fee          |
2022-05-21 08:00:00| 3196776.081081081| 0.09249903756756757|
2022-05-21 09:00:00|2968309.7441860465| 0.08610186255813955|
2022-05-21 10:00:00|2991568.2954545454| 0.08841007795454545|
2022-05-21 11:00:00| 3055516.085106383| 0.09578694297872341|
2022-05-21 12:00:00|3074216.8214285714| 0.09270591125000001|
```

</tab>

</Terminal>

![区块重量和费用的图表](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/weight_fees.png) 

你可以看到区块重量和区块挖矿费用确实是紧密相连的。实际上，你还可以看到这个图表上还有增长的空间，它们可以包含更多的交易。

### 每个区块的平均矿工收入是多少？

现在，分析矿工通过挖矿新区块在区块链上实际产生的收入，包括费用和区块奖励。这个查询分析了最近一天的数据，并使用了12小时移动平均值。

<Terminal>

<tab label='SQL'>

```sql
SELECT
   bucket as "time",
   average_y(rolling(stats_miner_revenue) OVER (ORDER BY bucket RANGE '12 hours' PRECEDING))*0.00000001 AS "revenue in BTC"
FROM one_hour_coinbase
WHERE bucket > NOW() - INTERVAL '1 day'
ORDER BY 1
```

</tab>

<tab label="数据">

```
time               |revenue in BTC    |
2022-05-21 08:00:00| 6.342499037567568|
2022-05-21 09:00:00|  6.33610186255814|
2022-05-21 10:00:00| 6.338410077954546|
2022-05-21 11:00:00| 6.345786942978723|
2022-05-21 12:00:00|     6.34270591125|
```

</tab>

</Terminal>

![最近一天每个区块的平均矿工收入的图表](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/miner_revenue_per_block.png) 

为了使图表更有趣，将BTC-USD汇率加入分析，并扩大时间范围：

<Terminal>

<tab label='SQL'>

```sql
SELECT
   bucket as "time",
   average_y(rolling(stats_miner_revenue) OVER (ORDER BY bucket RANGE '12 hours' PRECEDING))*0.00000001 AS "revenue in BTC",
    average_x(rolling(stats_miner_revenue) OVER (ORDER BY bucket RANGE '12 hours' PRECEDING)) AS "revenue in USD"
FROM one_hour_coinbase
WHERE bucket > NOW() - INTERVAL '1 day'
ORDER BY 1
```

</tab>

<tab label="数据">

```
time               |revenue in BTC    |revenue in USD    |
2022-05-21 08:00:00| 6.342499037567568|187416.12836756755|
2022-05-21 09:00:00|  6.33610186255814|187001.94948139536|
2022-05-21 10:00:00| 6.338410077954546| 187037.7794659091|
2022-05-21 11:00:00| 6.345786942978723|187166.63164042553|
2022-05-21 12:00:00|     6.34270591125|186870.74608750004|
```

</tab>

</Terminal>

![最近五天每个区块的平均矿工收入的图表，以BTC和USD计](https://assets.timescale.com/docs/images/tutorials/bitcoin-blockchain/miner_revenue_per_block_with_btcusd.png)

[docs-cagg]: /use-timescale/:currentVersion:/continuous-aggregates/
[docs-hyperfunctions]: /use-timescale/:currentVersion:/hyperfunctions/
[install-toolkit]: /self-hosted/:currentVersion:/tooling/install-toolkit/
[stats_agg]: /api/:currentVersion:/hyperfunctions/statistical-and-regression-analysis/stats_agg-one-variable/
