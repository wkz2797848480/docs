---
标题: 查询比特币区块链-查询数据
摘要: 查询比特币区块链数据集。
产品: [云服务]
关键词: [初学者，加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 查询比特币区块链
---

# 查询数据

当您加载了数据集后，可以开始构建一些查询来发现数据所揭示的信息。在本节中，您将学习如何编写回答以下问题：

*   [最近五笔coinbase交易是什么？](#最近五笔coinbase交易是什么)
*   [最近五笔交易是什么？](#最近五笔交易是什么)
*   [最近五个区块是什么？](#最近五个区块是什么)

## 最近五笔coinbase交易是什么？

在上一个步骤中，您从结果中排除了coinbase交易。[Coinbase][coinbase-def]交易是区块中的第一笔交易，它们包含了矿工因挖掘货币而获得的奖励。要找出最近的coinbase交易，您可以使用类似的`SELECT`语句，但搜索coinbase类型的交易。如果您再次包含交易的美元价值，您会注意到每笔交易的价值都是0美元。这是因为在coinbase交易中，货币尚未转移所有权。

<Procedure>

### 查找最近五笔coinbase交易

1.  连接到包含比特币数据集的Timescale数据库。
2.  在psql提示符下，使用此查询选择最近的五笔coinbase交易：

    ```sql
    SELECT time, hash, block_id, fee_usd FROM transactions
    WHERE is_coinbase IS TRUE
    ORDER BY time DESC
    LIMIT 5;
    ```

3.  您得到的数据看起来像这样：

    ```sql
                 time          |                               hash                               | block_id | fee_usd
    ------------------------+------------------------------------------------------------------+----------+---------
     2023-06-12 23:54:18+00 | 22e4610bc12d482bc49b7a1c5b27ad18df1a6f34256c16ee7e499b511e02d71e |   794111 |       0
     2023-06-12 23:53:08+00 | dde958bb96a302fd956ced32d7b98dd9860ff82d569163968ecfe29de457fedb |   794110 |       0
     2023-06-12 23:44:50+00 | 75ac1fa7febe1233ee57ca11180124c5ceb61b230cdbcbcba99aecc6a3e2a868 |   794109 |       0
     2023-06-12 23:44:14+00 | 1e941d66b92bf0384514ecb83231854246a94c86ff26270fbdd9bc396dbcdb7b |   794108 |       0
     2023-06-12 23:41:08+00 | 60ae50447254d5f4561e1c297ee8171bb999b6310d519a0d228786b36c9ffacf |   794107 |       0
    (5 rows)
    ```

</Procedure>

## 最近五笔交易是什么？

这个数据集包含了最近五天的比特币交易。要找出数据集中最近的交易，您可以使用`SELECT`语句。在这种情况下，您想要找到非coinbase交易，按时间降序排序，并获取前五个结果。您还希望看到区块ID以及交易的美元价值。

<Procedure>

### 查找最近五笔交易

1.  连接到包含比特币数据集的Timescale数据库。
2.  在psql提示符下，使用此查询选择最近的五笔非coinbase交易：

    ```sql
    SELECT time, hash, block_id, fee_usd FROM transactions
    WHERE is_coinbase IS NOT TRUE
    ORDER BY time DESC
    LIMIT 5;
    ```

3.  您得到的数据看起来像这样：

    ```sql
                  time          |                               hash                               | block_id | fee_usd
    ------------------------+------------------------------------------------------------------+----------+---------
     2023-06-12 23:54:18+00 | 6f709d52e9aa7b2569a7f8c40e7686026ede6190d0532220a73fdac09deff973 |   794111 |   7.614
     2023-06-12 23:54:18+00 | ece5429f4a76b1603aecbee31bf3d05f74142a260e4023316250849fe49115ae |   794111 |   9.306
     2023-06-12 23:54:18+00 | 54a196398880a7e2e38312d4285fa66b9c7129f7d14dc68c715d783322544942 |   794111 | 13.1928
     2023-06-12 23:54:18+00 | 3e83e68735af556d9385427183e8160516fafe2f30f30405711c4d64bf0778a6 |   794111 |  3.5416
     2023-06-12 23:54:18+00 | ca20d073b1082d7700b3706fe2c20bc488d2fc4a9bb006eb4449efe3c3fc6b2b |   794111 |  8.6842
    (5 rows)
    ```

</Procedure>

## 最近五个区块是什么？

在这个步骤中，您使用一个更复杂的查询来返回最近的五个区块，并展示每个区块的一些额外信息，包括区块重量、每个区块中的交易数量以及区块总价值（以美元计）。

<Procedure>

### 查找最近五个区块

1.  连接到包含比特币数据集的Timescale数据库。
2.  在psql提示符下，使用此查询选择最近的五个区块：

    ```sql
    WITH recent_blocks AS (
     SELECT block_id FROM transactions
     WHERE is_coinbase IS TRUE
     ORDER BY time DESC
     LIMIT 5
    )
    SELECT
     t.block_id, count(*) AS transaction_count,
     SUM(weight) AS block_weight,
     SUM(output_total_usd) AS block_value_usd
    FROM transactions t
    INNER JOIN recent_blocks b ON b.block_id = t.block_id
    WHERE is_coinbase IS NOT TRUE
    GROUP BY t.block_id;
    ```

3.  您得到的数据看起来像这样：

    ```sql
     block_id | transaction_count | block_weight |  block_value_usd
    ----------+-------------------+--------------+--------------------
       794108 |              5625 |      3991408 |  65222453.36381342
       794111 |              5039 |      3991748 |  5966031.481099684
       794109 |              6325 |      3991923 |  5406755.801599815
       794110 |              2525 |      3995553 |  177249139.6457974
       794107 |              4464 |      3991838 | 107348519.36559173
    (5 rows)
    ```

</Procedure>

[coinbase-def]: https://www.pcmag.com/encyclopedia/term/coinbase-transaction
