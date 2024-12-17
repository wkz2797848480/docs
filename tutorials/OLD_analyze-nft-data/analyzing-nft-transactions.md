---
标题: 分析非同质化代币（NFT）交易
摘要: 分析来自 OpenSea（一个 NFT 交易平台）市场的 NFT 交易情况。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [加密货币，区块链，金融，分析]
标签: [NFT]
布局组件: [大尺寸的上一篇 / 下一篇按钮]
内容分组: 分析 NFT 数据
---

# 分析NFT交易

当您成功收集并导入数据后，是时候进行分析了。本次分析使用的是我们的导入脚本收集的数据，其中仅包含2021年1月1日至2021年10月12日在OpenSea市场上发生的成功销售交易，如OpenSea API所报告的。

为简化起见，本教程仅分析使用`ETH`作为支付符号的交易，但如果您愿意，可以修改脚本以在分析中包含更多的支付符号。

本节的所有查询以及一些额外的查询都在我们的[NFT入门工具包在GitHub][nft-starter-kit]中的[`queries.sql`文件][queries]。

我们将分析分为两部分：简单查询和复杂查询。但首先，我们创建一些加速查询的东西：TimescaleDB连续聚合。

<Highlight type="note">
本节中的所有查询仅包括可以从OpenSea API获取的数据。
</Highlight>

## 使用连续聚合加速查询

TimescaleDB连续聚合加速需要处理大量数据的工作负载。它们看起来像PostgreSQL物化视图，但具有内置的刷新策略，确保新数据进来时数据是最新的。此外，刷新过程谨慎地只刷新物化视图中实际需要更改的数据，从而避免了未更改数据的重新计算。这种智能刷新过程大大提高了物化视图的刷新性能，刷新策略确保数据始终是最新的。

[连续聚合][cont-agg]通常用于加速仪表板和可视化，汇总高频率采样的数据，并在长时间周期内查询降采样的数据。

本教程创建两个连续聚合以加速对资产和收藏的查询。

### 资产连续聚合

创建一个名为`assets_daily`的新连续聚合，计算并存储有关每天所有资产的以下信息：`asset_id`、它所属的收藏、`日平均价格`、`中位数价格`、`销售量`、`ETH量`、`开盘`、`最高`、`最低`和`收盘价`：

```sql
/* 资产连续聚合 */
CREATE MATERIALIZED VIEW assets_daily
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 day', time) AS bucket,
asset_id,
collection_id,
mean(percentile_agg(total_price)) AS mean_price,
approx_percentile(0.5, percentile_agg(total_price)) AS median_price,
COUNT(*) AS volume,
SUM(total_price) AS volume_eth,
FIRST(total_price, time) AS open_price,
MAX(total_price) AS high_price,
MIN(total_price) AS low_price,
LAST(total_price, time) AS close_price
FROM nft_sales
WHERE payment_symbol = 'ETH'
GROUP BY bucket, asset_id, collection_id
```

添加一个刷新策略，以最新的数据每日更新连续聚合，以便您可以在查询时节省计算：

```sql
SELECT add_continuous_aggregate_policy('assets_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

### 收藏连续聚合

创建另一个名为`collections_daily`的连续聚合，计算并存储有关每天所有收藏的以下信息，包括`日平均价格`、`中位数价格`、`销售量`、`ETH量`、`最昂贵的NFT`和`最高价格`：

```sql
/* 收藏连续聚合 */
CREATE MATERIALIZED VIEW collections_daily
WITH (timescaledb.continuous) AS
SELECT
collection_id,
time_bucket('1 day', time) AS bucket,
mean(percentile_agg(total_price)) AS mean_price,
approx_percentile(0.5, percentile_agg(total_price)) AS median_price,
COUNT(*) AS volume,
SUM(total_price) AS volume_eth,
LAST(asset_id, total_price) AS most_expensive_nft_id,
MAX(total_price) AS max_price
FROM nft_sales
GROUP BY bucket, collection_id;

/* 刷新策略 */
SELECT add_continuous_aggregate_policy('collections_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day');
```

当您提出可以用日聚合帮助回答的问题时，可以查询连续聚合，而不是`nft_sales`超表中的原始数据。这有助于加快结果的速度。

## 简单查询

您可以开始分析关于2021年发生的NFT销售的简单问题，并使用SQL查询来回答它们。将这些查询作为您自己进一步分析的起点。您可以修改每个查询，以分析您感兴趣的时间段、资产、收藏或账户！

尽可能地，我们包括了Superset的仪表板示例，作为创建您自己的仪表板的灵感，该仪表板使用免费、开源工具监控和分析NFT销售。您可以在[NFT入门工具包Github仓库][nft-starter-kit]中找到用于创建每个图表的代码。

### 销售量最高的收藏

哪些收藏的销售量最高？回答这个问题是寻找资产经常被交易的收藏的好起点，这对于考虑NFT转售价值的买家来说很重要。如果您在以下收藏之一中购买NFT，您很有可能会找到买家。在此查询中，您按总销售量对收藏进行排序，但您也可以按ETH量排序：

```sql
/* 销售量最高的收藏？ */
SELECT
slug,
SUM(volume) total_volume,
SUM(volume_eth) total_volume_eth
FROM collections_daily cagg
INNER JOIN collections c ON cagg.collection_id = c.id
GROUP BY cagg.collection_id, slug
ORDER BY total_volume DESC;
```

| slug               | total_volume | total_volume_eth   |
|--------------------|--------------|--------------------|
| sorare             | 339776       | 35113.062124036835 |
| rarible            | 87594        | 41663.18012651946  |
| art-blocks-factory | 45861        | 43607.73207320631  |
| axie               | 43074        | 6692.242340266918  |
| cryptokitties      | 41300        | 5560.907800845506  |
| parallelalpha      | 36892        | 31212.686399159273 |
| art-blocks         | 35976        | 199016.27793424827 |
| ape-gang           | 25782        | 4663.009300672081  |
| 24px               | 24872        | 3203.9084810874024 |
| pudgypenguins      | 24165        | 35949.81731415086  |

对于这个查询，您利用了存储在`collections_daily`连续聚合中的关于收藏的预计算数据。您还执行了一个`INNER JOIN`在收藏关系表上，以找到以人类可读形式表示的收藏名称，由`slug`表示。

从连续聚合中查询更快，并且可以编写更短、更易读的查询。这是一个您将在本教程中再次使用的模式，所以要留意！

### 某个收藏的日常销售

对于某个特定收藏，每天发生了多少销售？此查询查看`cryptokitties`收藏中NFT的日常销售量。这可以帮助您找出NFT交易者更活跃的日子，并帮助您发现关于一周中哪一天或一个月中哪一天的销售量更高或更低以及原因的模式。

您可以修改此查询，以查看您最喜欢的NFT收藏，例如`cryptopunks`、`lazy-lions`或`afrodroids-by-owo`：

```sql
SELECT bucket, slug, volume
FROM collections_daily cagg
INNER JOIN collections c ON cagg.collection_id = c.id
WHERE slug = 'cryptokitties'
ORDER BY bucket DESC;
```

bucket             |slug         |volume|
-------------------|-------------|------|
2021-10-12 02:00:00|cryptokitties|    48|
2021-10-11 02:00:00|cryptokitties|    61|
2021-10-10 02:00:00|cryptokitties|    84|
2021-10-09 02:00:00|cryptokitties|    73|
2021-10-08 02:00:00|cryptokitties|    56|
...

以下是此查询在Apache Superset中的时间序列图表的样子：

![每日NFT交易数量](https://assets.timescale.com/docs/images/tutorials/nft-tutorial/daily-number-of-nft-transactions.jpg) 

提醒一下，像这样的图表是预先构建好的，并且可以作为我们[NFT入门工具包][nft-starter-kit]中预构建仪表板的一部分，供您使用和修改。

### 不同收藏的日常NFT销售比较

一个收藏的NFT日常销售与另一个收藏相比如何？此查询比较了两个流行的NFT收藏：CryptoKitties和Ape Gang在过去三个月的日常销售：

```sql
/* 过去3个月，“CryptoKitties”与Ape Gang的日常NFT交易数量？ */
SELECT bucket, slug, volume
FROM collections_daily cagg
INNER JOIN collections c ON cagg.collection_id = c.id
WHERE slug IN ('cryptokitties', 'ape-gang') AND bucket > NOW() - INTERVAL '3 month'
ORDER BY bucket DESC, slug;
```

bucket             |slug         |volume|
-------------------|-------------|------|
2021-10-12 02:00:00|ape-gang
     |    58|
2021-10-12 02:00:00|cryptokitties|    48|
2021-10-11 02:00:00|ape-gang     |   208|
2021-10-11 02:00:00|cryptokitties|    61|
2021-10-10 02:00:00|ape-gang     |   248|
2021-10-10 02:00:00|cryptokitties|    84|
...

![不同收藏的比较](https://assets.timescale.com/docs/images/tutorials/nft-tutorial/comparison-of-different-collections.jpg) 

这种类型的查询有助于跟踪您感兴趣的或拥有资产的收藏的销售活动，以便您可以看到其他NFT持有者的活动。此外，您可以修改考虑的时间段，以查看更大（如9个月）或更小（如14天）的时间段。

### Snoop Dogg的NFT活动（或个人账户活动）

特定人在一定时期内购买了多少NFT？这类查询有助于监控流行NFT收藏家的活动，比如美国说唱歌手Snoop Dogg（或[Cozomo_de_Medici][snoop-dogg-opensea]）或非洲NFT传道者[Daliso Ngoma][daliso-opensea]，甚至比较多个收藏家交易模式。由于NFT交易在以太坊区块链上是公开的，我们的数据库包含卖家（`seller_account`）和买家（`winner_account`）列，因此您可以分析特定账户的购买活动。

这个查询使用Snoop Dogg的地址来分析他的交易，但您可以编辑查询，在`WHERE`子句中添加任何地址，以查看指定账户的交易：

```sql
/* Snoop Dogg在过去3个月的交易汇总 */
WITH snoop_dogg AS (
    SELECT id FROM accounts
    WHERE address = '0xce90a7949bb78892f159f428d0dc23a8e3584d75'
)
SELECT
COUNT(*) AS trade_count,
COUNT(DISTINCT asset_id) AS nft_count,
COUNT(DISTINCT collection_id) AS collection_count,
COUNT(*) FILTER (WHERE seller_account = (SELECT id FROM snoop_dogg)) AS sale_count,
COUNT(*) FILTER (WHERE winner_account = (SELECT id FROM snoop_dogg)) AS buy_count,
SUM(total_price) AS total_volume_eth,
AVG(total_price) AS avg_price,
MIN(total_price) AS min_price,
MAX(total_price) AS max_price
FROM nft_sales
WHERE payment_symbol = 'ETH' AND ( seller_account = (SELECT id FROM snoop_dogg) OR winner_account = (SELECT id FROM snoop_dogg) )
AND time > NOW()-INTERVAL '3 months'
```

trade_count|nft_count|collection_count|sale_count|buy_count|total_volume_eth  |avg_price         |min_price|max_price|
-----------|---------|----------------|----------|---------|------------------|------------------|---------|---------|
        59|       57|              20|         1|       58|1835.5040000000006|31.110237288135604|      0.0|   1300.0|

从查询结果中，我们可以看到Snoop Dogg在过去3个月总共进行了59次交易（购买了58次，仅出售了1次）。他的交易包括57个单独的NFT和23个收藏，总共花费了1835.504 ETH，最低支付价格为0，最高为1300 ETH。

### 某个收藏中最昂贵的资产

某个特定收藏中最昂贵的NFT是什么？这个查询查看特定收藏（CryptoKitties）并找到从其中售出的最昂贵的NFT。这可以帮助您找到收藏中的稀有物品，并查看使其稀有的属性，以便您购买具有类似属性的物品：

```sql
/* CryptoKitties收藏中最昂贵的5个NFT */
SELECT a.name AS nft, total_price, time, a.url  FROM nft_sales s
INNER JOIN collections c ON c.id = s.collection_id
INNER JOIN assets a ON a.id = s.asset_id
WHERE slug = 'cryptokitties' AND payment_symbol = 'ETH'
ORDER BY total_price DESC
LIMIT 5
```

nft            |total_price|time               |url                                                                    |
---------------|-----------|-------------------|-----------------------------------------------------------------------|
Founder Cat #40|      225.0|2021-09-03 14:59:16|[链接](https://opensea.io/assets/0x06012c8cf97bead5deae237070f9587f8e7a266d/40)| 
Founder Cat #17|      177.0|2021-09-03 01:58:13|[链接](https://opensea.io/assets/0x06012c8cf97bead5deae237070f9587f8e7a266d/17)| 
润龙🐱‍👓创世猫王44# |      150.0|2021-09-03 02:01:11|[链接](https://opensea.io/assets/0x06012c8cf97bead5deae237070f9587f8e7a266d/44)| 
grey           |      149.0|2021-09-03 02:32:26|[链接](https://opensea.io/assets/0x06012c8cf97bead5deae237070f9587f8e7a266d/16)| 
Founder Cat #38|      148.0|2021-09-03 01:58:13|[链接](https://opensea.io/assets/0x06012c8cf97bead5deae237070f9587f8e7a266d/38)| 

### 某个收藏中资产的日常ETH量

特定收藏的日常以太坊（ETH）量是多少？使用CryptoKitties作为例子，此查询计算在一定收藏中NFT销售的日常总ETH花费：

```sql
/* CryptoKitties NFT交易的日常ETH量？ */
SELECT bucket, slug, volume_eth
FROM collections_daily cagg
INNER JOIN collections c ON cagg.collection_id = c.id
WHERE slug = 'cryptokitties'
ORDER BY bucket DESC;
```

bucket             |slug         |volume_eth         |
-------------------|-------------|-------------------|
2021-10-12 02:00:00|cryptokitties| 1.6212453906698892|
2021-10-11 02:00:00|cryptokitties| 1.8087566697786246|
2021-10-10 02:00:00|cryptokitties|  2.839395250444516|
2021-10-09 02:00:00|cryptokitties|  4.585460691370447|
2021-10-08 02:00:00|cryptokitties|   5.36784615406771|
2021-10-07 02:00:00|cryptokitties| 16.591879406085422|
2021-10-06 02:00:00|cryptokitties| 11.390538587035808|
...

![资产的日常ETH量](https://assets.timescale.com/docs/images/tutorials/nft-tutorial/daily-eth-volume-of-assets.jpg) 

<Highlight type="note">
此图表使用对数刻度，您可以在Superset中图表设置中配置。
</Highlight>

### 多个收藏的日常ETH量比较

一个收藏中资产的日常ETH花费量与其他收藏相比如何？此查询使用CryptoKitties和Ape Gang作为例子，找出过去三个月这些收藏中购买资产的日常ETH花费量。您可以扩展此查询，监控并与您最喜欢的NFT收藏的日常花费量进行比较，并发现销售模式：

```sql
/* NFT交易的日常ETH量：CryptoKitties与Ape Gang？ */
SELECT bucket, slug, volume_eth
FROM collections_daily cagg
INNER JOIN collections c ON cagg.collection_id = c.id
WHERE slug IN ('cryptokitties', 'ape-gang') AND bucket > NOW() - INTERVAL '3 month'
ORDER BY bucket, slug DESC;
```

bucket             |slug         |volume_eth        |
-------------------|-------------|------------------|
2021-10-12 02:00:00|ape-gang     | 54.31030000000001|
2021-10-12 02:00:00|cryptokitties|1.6212453906698896|
2021-10-11 02:00:00|ape-gang     |205.19786218340954|
2021-10-11 02:00:00|cryptokitties|1.8087566697786257|
2021-10-10 02:00:00|ape-gang     | 240.0944201232798|
2021-10-10 02:00:00|cryptokitties| 2.839395250444517|
...

![不同收藏的日常ETH量比较](https://assets.timescale.com/docs/images/tutorials/nft-tutorial/comparison-daily-eth-volume-collections.jpg) 

<Highlight type="note">
上述图表使用对数刻度，我们在Superset中图表设置中进行了配置。
</Highlight>

### 某个收藏中资产的日常平均和中位数销售价格

当您分析特定收藏中资产的日常价格时，两个有用的统计数据是平均价格和中位数价格。此查询找到CryptoKitties收藏中资产的日常平均和中位数销售价格：

```sql
/* CryptoKitties的平均与中位数销售价格？ */
SELECT bucket, slug, mean_price, median_price
FROM collections_daily cagg
INNER JOIN collections c ON cagg.collection_id = c.id
WHERE slug = 'cryptokitties'
ORDER BY
 bucket DESC;
```

bucket             |slug         |mean_price          |median_price         |
-------------------|-------------|--------------------|---------------------|
2021-10-12 02:00:00|cryptokitties| 0.03377594563895602|  0.00600596459124994|
2021-10-11 02:00:00|cryptokitties|0.029651748684895486| 0.008995758681494385|
2021-10-10 02:00:00|cryptokitties| 0.03380232441005376|  0.00600596459124994|
2021-10-09 02:00:00|cryptokitties| 0.06281453001877325| 0.010001681651251936|
2021-10-08 02:00:00|cryptokitties| 0.09585439560835196| 0.010001681651251936|
...

由于计算大数据集的平均值和中位数在计算上成本较高，我们使用了[`percentile_agg`超函数][percentile-agg]，这是Timescale Toolkit扩展的一部分。它准确地近似了这两个统计数据，如我们在本教程前面创建的连续聚合中`mean_price`和`median_price`的定义所示：

```sql
CREATE MATERIALIZED VIEW collections_daily
WITH (timescaledb.continuous) AS
SELECT
collection_id,
time_bucket('1 day', time) AS bucket,
mean(percentile_agg(total_price)) AS mean_price,
approx_percentile(0.5, percentile_agg(total_price)) AS median_price,
COUNT(*) AS volume,
SUM(total_price) AS volume_eth,
LAST(asset_id, total_price) AS most_expensive_nft,
MAX(total_price) AS max_price
FROM nft_sales s
GROUP BY bucket, collection_id;
```

### 顶级买家的日常总成交量

最活跃的账户在哪些日子购买？要回答这个问题，您可以分析基于NFT购买数量的前五大NFT买家账户，以及他们随时间购买NFT的日成交量。这是深入分析的好起点，因为它可以帮助您找到导致这些用户购买大量NFT的日子。例如，ETH价格下跌，导致汽油费降低，或高预期收藏的下降：

```sql
/* 5大顶级买家的日常总成交量 */
WITH top_five_buyers AS (
   SELECT winner_account FROM nft_sales
   GROUP BY winner_account
   ORDER BY count(*) DESC
   LIMIT 5
)
SELECT time_bucket('1 day', time) AS bucket, count(*) AS total_volume FROM nft_sales
WHERE winner_account IN (SELECT winner_account FROM top_five_buyers)
GROUP BY bucket
ORDER BY bucket DESC
```

![顶级买家的成交量](https://assets.timescale.com/docs/images/tutorials/nft-tutorial/volume-top-buyers.jpg) 

## 复杂查询

让我们来看一些关于NFT数据集的更复杂的问题，以及更复杂的查询，以检索有趣的事物。

### 计算昨天最高交易量NFT的30分钟平均和中位数销售价格

过去一天中，最高交易量NFT的平均和中位数销售价格是多少，以30分钟为间隔？

```sql
/* 计算2021-10-17最高交易量NFT的15分钟平均和中位数销售价格 */
WITH one_day AS (
   SELECT time, asset_id, total_price FROM nft_sales
   WHERE time >= '2021-10-17' AND time < '2021-10-18' AND payment_symbol = 'ETH'
)
SELECT time_bucket('30 min', time) AS bucket,
assets.name AS nft,
mean(percentile_agg(total_price)) AS mean_price,
approx_percentile(0.5, percentile_agg(total_price)) AS median_price
FROM one_day
INNER JOIN assets ON assets.id = one_day.asset_id
WHERE asset_id = (SELECT asset_id FROM one_day GROUP BY asset_id ORDER BY count(*) DESC LIMIT 1)
GROUP BY bucket, nft
ORDER BY bucket DESC;
```

bucket             |nft           |mean_price         |median_price        |
-------------------|--------------|-------------------|--------------------|
2021-10-17 23:30:00|Zero [Genesis]|               0.06| 0.06002456177152414|
2021-10-17 23:00:00|Zero [Genesis]|              0.118|  0.1180081944620535|
2021-10-17 22:30:00|Zero [Genesis]|       0.0785333333| 0.06002456177152414|
2021-10-17 22:00:00|Zero [Genesis]|             0.0775| 0.09995839119153871|
2021-10-17 21:30:00|Zero [Genesis]|             0.0555| 0.05801803032917102|

这是一个更复杂的查询，它使用PostgreSQL公用表表达式（CTE）首先创建一个过去一天的数据子表，称为`one_day`。然后您使用超函数time_bucket创建我们数据的30分钟桶，并使用[percentile_agg超函数][percentile-agg]来找到每个间隔周期的平均和中位数价格。最后，您联结`assets`表以获取特定NFT的名称，以便返回每个时间间隔的平均和中位数价格。

### 每个资产的日常OHLCV数据

开盘-最高-最低-收盘-成交量（OHLCV）图表通常用于说明金融工具的价格，最常见的是股票，随时间变化。您可以为单个NFT创建OHLCV图表，或获取一组NFT的OHLCV值。

此查询找到一天内销售超过100次的NFT的OHLCV，以及交易发生的日期：

```sql
/* 每个资产的日常OHLCV */
SELECT time_bucket('1 day', time) AS bucket, asset_id,
FIRST(total_price, time) AS open_price, LAST(total_price, time) AS close_price,
MIN(total_price) AS low_price, MAX(total_price) AS high_price,
count(*) AS volume
FROM nft_sales
WHERE payment_symbol = 'ETH'
GROUP BY bucket, asset_id
HAVING count(*) > 100
ORDER BY bucket
LIMIT 5;
```

bucket             |asset_id|open_price|close_price|low_price  |high_price|volume|
-------------------|--------|----------|-----------|-----------|----------|------|
2021-02-03 01:00:00|17790698|      0.56|       1.25|       0.07|       7.0|   148|
2021-02-05 01:00:00|17822636|       7.0|        0.7|        0.7|       8.4|   132|
2021-02-11 01:00:00|17927258|       0.8|        0.2|        0.1|       2.0|   103|
2021-02-26 01:00:00|18198072|       0.1|        0.1|        0.1|       0.1|   154|
2021-02-26 01:00:00|18198081|      0.25|       0.25|       0.25|      0.25|   155|

在此查询中，您使用了TimescaleDB超函数[`first()`][first-docs]和[`last()`][last-docs]来分别找到开盘和收盘价。这些超函数允许您通过按另一个列排序，对他们的组执行顺序扫描来找到一列的值。在这种情况下，您获得了`total_price`列的首个和最后一个值，按`time`列排序。[查看文档了解更多信息。][first-docs]

如果您想定期运行此查询，可以为其创建一个连续聚合，这将大大提高查询性能。此外，您可以去掉`LIMIT 5`并替换为一个额外的WHERE子句，用于过滤特定时间段，使查询更有用。

### 具有最大日内价格变动的资产

哪些资产具有最大的日内销售价格变动？您可以识别有趣的行为，比如一个资产被购买后在同一天内以更高（或更低）的价格再次出售。这有助于您识别NFT的好翻转，或者可能因为成为他们收藏的一部分而提升NFT价格的所有者品牌。

此查询找到过去六个月内具有最大日内销售价格变动的资产：

```sql
/* 过去6个月内按最大日内价格变动排序的资产 */
WITH top_assets AS (
 SELECT time_bucket('1 day', time) AS bucket, asset_id,
 FIRST(total_price, time) AS open_price, LAST(total_price, time) AS close_price,
 MAX(total_price)-MIN(total_price) AS intraday_max_change
 FROM nft_sales s
 WHERE payment_symbol = 'ETH' AND time > NOW() - INTERVAL '6 month'
 GROUP BY bucket, asset_id
 ORDER BY intraday_max_change DESC
 LIMIT 5
)
SELECT bucket, nft, url,   
 open_price, close_price,
 intraday_max_change
FROM top_assets ta
INNER JOIN LATERAL (
 SELECT name AS nft, url FROM assets a
 WHERE a.id = ta.asset_id
) assets ON TRUE;
```

<!-- markdown-link-check-disable -->
<!-- vale Google.Units = NO -->

bucket             |nft           |url                                                                           |open_price|close_price|intraday_max_change|
-------------------|--------------|------------------------------------------------------------------------------|----------|-----------|-------------------|
2021-09-22 02:00:00|Page          |[链接](https://opensea.io/assets/0xa7206d878c5c3871826dfdb42191c49b1d11f466/1)         |      0.72|     0.9999|           239.2889|
2021-09-23 02:00:00|Page          |[链接](https://opensea.io/assets/0xa7206d878c5c3871826dfdb42191c49b1d11f466/1)         |    0.9999|       1.14|              100.0|
2021-09-27 02:00:00|Skulptuur #647|[链接](https://opensea.io/assets/0xa7d8d9ef8d8ce8992df33d8b8cf4aebabd5bd270/173000647)|       25.0|       90.0|               65.0|
2021-09-25 02:00:00|Page          |[链接](https://opensea.io/assets/0xa7206d878c5c3871826dfdb42191c49b1d11f466/1)         |      1.41|      1.475|               61.3|
2021-09-26 02:00:00|Page          |[链接](https://opensea.io/assets/0xa7206d878c5c3871826dfdb42191c49b1d11f466/1)         |      1.48|      4.341|              43.05|

<!-- vale Google.Units = YES -->
<!-- markdown-link-check-enable -->

## 资源和后续步骤

本节包含完成本教程后要执行的操作信息，以及一些更多资源的链接。

### 领取您的限量版时间旅行老虎NFT

首批完成本教程的20人可以免费获得[时间旅行老虎收藏][eon-collection]中的限量版NFT！

现在您已经完成了本教程，您所需要做的就是在[这个表格][nft-form]中回答问题（包括挑战问题），我们将把限量版Eon NFT发送到您的ETH地址（无需您承担任何费用！）。

您可以在[OpenSea][eon-collection]上查看时间旅行老虎收藏中的所有NFT。

### 基于NFT入门工具包构建

恭喜！您现在已经可以使用NFT数据和TimescaleDB了。查看我们的[NFT入门工具包][nft-starter-kit]，作为您构建自己更复杂的NFT分析项目的起点。

入门工具包包含：

*   一个数据导入脚本，从OpenSea实时收集数据并导入到TimescaleDB
*   一个样本数据集，如果您不想导入实时数据，可以快速开始
*   一个存储NFT销售、资产、收藏和所有者的模式
*   一个预加载了样本NFT数据的本地TimescaleDB数据库
*   在[Apache Superset][superset]和[Grafana][grafana]中预构建的仪表板和图表，用于可视化您的数据分析
*   作为您自己分析起点的查询

### 了解更多关于如何使用TimescaleDB存储和分析加密数据

查看这些资源，了解更多关于使用TimescaleDB与加密数据的信息：

*   [分析加密货币市场数据][analyze-cryptocurrency]
*   [使用PostgreSQL和TimescaleDB分析比特币、以太坊和其他4100多种加密货币][analyze-bitcoin]
*   [了解TimescaleDB用户Messari如何使用数据向所有人开放加密经济][messari]
*   [了解一个TimescaleDB用户如何构建一个成功的加密交易机器人][trading-bot]

[analyze-bitcoin]: https://blog.timescale.com/blog/analyzing-bitcoin-ethereum-and-4100-other-cryptocurrencies-using-postgresql-and-timescaledb/
[analyze-cryptocurrency]: /tutorials/:currentVersion:/blockchain-analyze/
[cont-agg]: /use-timescale/:currentVersion:/continuous-aggregates
[daliso-opensea]: https://opensea.io/daliso
[eon-collection]: https://opensea.io/collection/time-travel-tigers-by-timescale
[first-docs]: /api/:currentVersion:/hyperfunctions/first/
[grafana]: https://grafana.com
[last-docs]: /api/:currentVersion:/hyperfunctions/last
[messari]: https://blog.timescale.com/blog/how-messari-uses-data-to-open-the-cryptoeconomy-to-everyone/
[nft-form]: https://docs.google.com/forms/d/e/1FAIpQLSdZMzES-vK8K_pJl1n7HWWe5-v6D9A03QV6rys18woGTZr0Yw/viewform?usp=sf_link
[nft-starter-kit]: https://github.com/timescale/nft-starter-kit
[percentile-agg]: /api/:currentVersion:/hyperfunctions/percentile-approximation/uddsketch/#percentile_agg
[queries]: https://github.com/timescale/nft-starter-kit/blob/master/queries.sql
[snoop-dogg-opensea]: https://opensea.io/Cozomo_de_Medici
[superset]: https://superset.apache.org
[trading-bot]: https://blog.timescale.com/blog/how-i-power-a-successful-crypto-trading-bot-with-timescaledb/
