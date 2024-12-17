---
标题: 分析比特币区块链 —— 查询数据
摘要: 利用 Timescale 超级函数分析比特币区块链。
产品: [云服务]
关键词: [中级，加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析比特币区块链
---

# 分析数据

当您加载数据集后，可以创建一些连续聚合，并开始构建查询以发现数据所揭示的信息。本教程使用[Timescale超函数][about-hyperfunctions]构建在标准PostgreSQL中不可能的查询。

在本节中，您将学习如何编写回答以下问题：

*   [交易数量与交易费用之间是否存在联系？](#交易数量与交易费用之间是否存在联系)
*   [交易量是否影响BTC-USD汇率？](#交易量是否影响BTC-USD汇率)
*   [区块中更多的交易是否意味着区块更昂贵？](#区块中更多的交易是否意味着区块更昂贵)
*   [平均矿工收入中有多少百分比来自费用与区块奖励相比？](#平均矿工收入中有多少百分比来自费用与区块奖励相比)
*   [区块重量如何影响矿工费用？](#区块重量如何影响矿工费用)
*   [每个区块的平均矿工收入是多少？](#每个区块的平均矿工收入是多少)

## 创建连续聚合

您可以使用[连续聚合][docs-cagg]简化和加速查询。对于本教程，您需要三个连续聚合，关注数据集的三个方面：比特币交易、区块和coinbase交易。在每个连续聚合定义中，`time_bucket()`函数控制时间桶的大小。所有示例都使用1小时的时间桶。

<Procedure>

### 连续聚合：交易

1.  连接到包含比特币数据集的Timescale数据库。
2.  在psql提示符下，创建一个名为`one_hour_transactions`的连续聚合。此视图包含每小时交易的聚合数据：

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

3.  添加刷新策略以保持连续聚合最新：

    ```sql
    SELECT add_continuous_aggregate_policy('one_hour_transactions',
       start_offset => INTERVAL '3 hours',
       end_offset => INTERVAL '1 hour',
       schedule_interval => INTERVAL '1 hour');
    ```

4.  创建一个名为`one_hour_blocks`的连续聚合。此视图包含每小时开采的所有区块的聚合数据：

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

5.  添加刷新策略以保持连续聚合最新：

    ```sql
    SELECT add_continuous_aggregate_policy('one_hour_blocks',
       start_offset => INTERVAL '3 hours',
       end_offset => INTERVAL '1 hour',
       schedule_interval => INTERVAL '1 hour');
    ```

6.  创建一个名为`one_hour_coinbase`的连续聚合。此视图包含每小时矿工作为奖励收到的所有交易的聚合数据：

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

7.  添加刷新策略以保持连续聚合最新：

    ```sql
    SELECT add_continuous_aggregate_policy('one_hour_coinbase',
       start_offset => INTERVAL '3 hours',
       end_offset => INTERVAL '1 hour',
       schedule_interval => INTERVAL '1 hour');
    ```

</Procedure>
### 探索是否更高的区块重量意味着更高的挖矿成本

1. 连接到包含比特币数据集的Timescale数据库。
2. 在psql提示符下，使用以下查询返回区块重量与挖矿费用的比较：

    ```sql
    SELECT
       bucket as "time",
       avg(block_weight) as "block weight",
       avg(block_fee_sat*0.00000001) as "mining fee"
    FROM one_hour_blocks
    WHERE bucket > now() - INTERVAL '5 day'
    group by bucket
    ORDER BY 1;
    ```

3. 返回的数据看起来像这样：

    ```sql
              time          |     block weight     |       mining fee
    ------------------------+----------------------+------------------------
     2023-06-10 08:00:00+00 | 3992809.250000000000 | 0.29221418750000000000
     2023-06-10 09:00:00+00 | 3991766.333333333333 | 0.50512649666666666667
     2023-06-10 10:00:00+00 | 3992918.250000000000 | 0.44783255750000000000
     2023-06-10 11:00:00+00 | 3991873.000000000000 | 0.39303009500000000000
     2023-06-10 12:00:00+00 | 3992934.000000000000 | 0.25590717142857142857
    ...
    ```

4. <Optional />在Grafana中可视化这一点，创建一个新的面板，选择比特币数据集作为您的数据源，并输入上一步的查询。在`Format as`部分，选择`Time series`。
5. <Optional />要使这个可视化更有用，添加一个覆盖层，将费用放在不同的Y轴上。在选项面板中，为`mining fee`字段添加一个覆盖层，对于`Axis > Placement`选择`Right`。

    <img
    class="main-content__illustration"
    src="https://assets.timescale.com/docs/images/grafana-blockweight-miningfee.webp"
    width={1375} height={944}
    alt="可视化区块重量和挖矿费用"
    />

</Procedure>

## 矿工收入中来自费用与区块奖励的百分比是多少？

在之前的查询中，您看到当区块重量和交易量较高时，挖矿费用也较高。这个查询从不同的角度分析数据。矿工收入不仅由挖矿费用组成，还包括挖新块的区块奖励。这个奖励目前是6.25 BTC，并且每四年减半。这个查询查看矿工收入中有多少来自费用，与区块奖励相比。

如果您选择在Grafana中可视化查询，您可以看到大多数矿工收入实际上来自区块奖励。费用从未超过总收入的几个百分点。

<Procedure>

### 探索矿工收入中来自费用与区块奖励的百分比

1. 连接到包含比特币数据集的Timescale数据库。
2. 在psql提示符下，使用以下查询返回coinbase交易，以及区块费用和奖励：

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

3. 返回的数据看起来像这样：

    ```sql
              time          |          fees          |   reward
    ------------------------+------------------------+------------
     2023-06-10 08:00:00+00 | 0.28247062857142857143 | 6.25000000
     2023-06-10 09:00:00+00 | 0.50512649666666666667 | 6.25000000
     2023-06-10 10:00:00+00 | 0.44783255750000000000 | 6.25000000
     2023-06-10 11:00:00+00 | 0.39303009500000000000 | 6.25000000
     2023-06-10 12:00:00+00 | 0.25590717142857142857 | 6.25000000
    ...
    ```

4. <Optional />在Grafana中可视化这一点，创建一个新的面板，选择比特币数据集作为您的数据源，并输入上一步的查询。在`Format as`部分，选择`Time series`。
5. <Optional />要使这个可视化更有用，将系列堆叠到100%。在选项面板中，在`Graph styles`部分，对于`Stack series`选择`100%`。

    <img
    class="main-content__illustration"
    src="https://assets.timescale.com/docs/images/grafana-coinbase-revenue.webp"
    width={1375} height={944}
    alt="可视化coinbase收入来源"
    />

</Procedure>

## 区块重量如何影响矿工费用？

您已经发现区块中的交易越多，挖矿成本越高。在这个查询中，您询问区块重量是否也是如此？区块中的交易越多，其重量越大，因此区块重量和挖矿费用应该紧密相关。这个查询使用12小时移动平均值来计算随时间变化的区块重量和挖矿费用。

如果您选择在Grafana中可视化查询，您可以看到区块重量和区块挖矿费用紧密相连。实际上，您还可以看到四百万重量单位的大小限制。这意味着单个区块仍有增长空间，它们可以包含更多的交易。

<Procedure>

### 探索区块重量如何影响矿工费用

1. 连接到包含比特币数据集的Timescale数据库。
2. 在psql提示符下，使用以下查询返回区块重量，以及区块费用和奖励：

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
    ORDER BY 1;
    ```

3. 返回的数据看起来像这样：

    ```sql
              time          |    block weight    |     mining fee
    ------------------------+--------------------+-------------------
     2023-06-10 09:00:00+00 | 3991766.3333333335 |  0.5051264966666666
     2023-06-10 10:00:00+00 | 3992424.5714285714 | 0.47238710285714286
     2023-06-10 11:00:00+00 |            3992224 | 0.44353000909090906
     2023-06-10 12:00:00+00 |  3992500.111111111 | 0.37056557222222225
     2023-06-10 13:00:00+00 |         3992446.65 | 0.39728022799999996
    ...
    ```

4. <Optional />在Grafana中可视化这一点，创建一个新的面板，选择比特币数据集作为您的数据源，并输入上一步的查询。在`Format as`部分，选择`Time series`。
5. <Optional />要使这个可视化更有用，添加一个覆盖层，将费用放在不同的Y轴上。在选项面板中，为`mining fee`字段添加一个覆盖层，对于`Axis > Placement`选择`Right`。

    <img
    class="main-content__illustration"
    src="https://assets.timescale.com/docs/images/grafana-blockweight-rewards.webp"
    width={1375} height={944}
    alt="可视化区块重量和挖矿费用"
    />

</Procedure>

## 每个区块的平均矿工收入是多少？

在这个最终的查询中，您分析矿工通过在区块链上挖新块实际产生的收入，包括费用和区块奖励。为了使分析更有趣，添加比特币对美元的汇率，并扩大时间范围。

<Procedure>

### 探索每个区块的平均矿工收入

1. 连接到包含比特币数据集的Timescale数据库。
2. 在psql提示符下，使用以下查询返回
每个区块的平均矿工收入，使用12小时移动平均值：

    ```sql
    SELECT
       bucket as "time",
       average_y(rolling(stats_miner_revenue) OVER (ORDER BY bucket RANGE '12 hours' PRECEDING))*0.00000001 AS "revenue in BTC",
        average_x(rolling(stats_miner_revenue) OVER (ORDER BY bucket RANGE '12 hours' PRECEDING)) AS "revenue in USD"
    FROM one_hour_coinbase
    WHERE bucket > NOW() - INTERVAL '5 days'
    ORDER BY 1;
    ```

3. 返回的数据看起来像这样：

    ```sql
              time          |   revenue in BTC   |   revenue in USD
    ------------------------+--------------------+-------------------
     2023-06-09 14:00:00+00 |       6.6732841925 |        176922.1133
     2023-06-09 15:00:00+00 |  6.785046736363636 |  179885.1576818182
     2023-06-09 16:00:00+00 |       6.7252952905 | 178301.02735000002
     2023-06-09 17:00:00+00 |  6.716377454814815 |  178064.5978074074
     2023-06-09 18:00:00+00 |    6.7784206471875 |   179709.487309375
    ...
    ```

4. <Optional />在Grafana中可视化这一点，创建一个新的面板，选择比特币数据集作为您的数据源，并输入上一步的查询。在`Format as`部分，选择`Time series`。
5. <Optional />要使这个可视化更有用，添加一个覆盖层，将美元放在不同的Y轴上。在选项面板中，为`mining fee`字段添加一个覆盖层，对于`Axis > Placement`选择`Right`。

    <img
    class="main-content__illustration"
    src="https://assets.timescale.com/docs/images/grafana-blockweight-revenue.webp"
    width={1375} height={944}
    alt="随时间可视化区块收入"
    />

</Procedure>


[docs-cagg]: /use-timescale/:currentVersion:/continuous-aggregates/
[about-hyperfunctions]: https://docs.timescale.com/use-timescale/latest/hyperfunctions/about-hyperfunctions/
