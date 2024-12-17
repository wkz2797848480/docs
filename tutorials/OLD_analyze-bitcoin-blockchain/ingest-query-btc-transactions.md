---
标题: 插入并查询比特币交易
摘要: 在你的数据库中摄取并存储比特币区块链数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一篇 / 下一篇按钮]
内容分组: 分析比特币区块链
---

# 插入和查询比特币交易

本教程的这一部分提供了一个示例数据库模式，你可以使用它在TimescaleDB中摄取和存储比特币区块链数据。该模式仅包含一个名为`transactions`的表。

## 比特币交易数据字段

本教程的示例比特币数据集包含以下字段：

| 字段 | 描述 |
|---|---|
| time | 交易的时间戳 |
| block_id | 区块ID |
| hash | 交易的哈希ID |
| size | 交易的大小，单位为KB |
| weight | 交易的大小，单位为重量单位 |
| is_coinbase | 交易是否为区块中的第一笔交易，包括矿工奖励 |
| output_total | 交易的价值，单位为萨托希（sat） |
| output_total_usd | 交易的价值，单位为美元 |
| fee | 交易费用，单位为萨托希（sat） |
| fee_usd | 交易费用，单位为美元 |

## 表定义

创建一个名为`transactions`的表来保存比特币数据。运行以下查询：

```sql
CREATE TABLE transactions (
   time TIMESTAMPTZ,
   block_id INT,
   hash TEXT,
   size INT,
   weight INT,
   is_coinbase BOOLEAN,
   output_total BIGINT,
   output_total_usd DOUBLE PRECISION,
   fee BIGINT,
   fee_usd DOUBLE PRECISION,
   details JSONB
);
```

表模式包括上述所有字段，以及一个额外的JSONB列名为`details`。此列存储有关每笔交易的额外信息的JSONB字符串。本教程中不使用此列的数据，但你可以探索这些数据，并受到启发进行自己的分析。

使用[`create_hypertable()`][create_hypertable]函数将表转换为超表。超表通过使用TimescaleDB的后台分块功能，为您提供性能提升。这个函数需要两个参数：表名和TIMESTAMP列的名称。在这种情况下，名称是`transactions`和`time`。

```sql
SELECT create_hypertable('transactions', 'time');
```

接下来，在超表上创建一些额外的索引。这优化了后续SQL查询的执行。

## 创建索引

当你创建一个超表时，TimescaleDB会自动在时间戳列上添加一个B树索引。这提高了按时间列过滤的查询性能。

为了加速使用`hash`列搜索单个交易的查询，向该列添加一个`HASH INDEX`：

```sql
CREATE INDEX hash_idx ON public.transactions USING HASH (hash)
```

接下来，通过在`block_id`列上添加索引来加速区块级别的查询：

```sql
CREATE INDEX block_idx ON public.transactions (block_id)
```

为确保你不会意外插入重复记录，在`time`和`hash`列上添加一个`UNIQUE INDEX`。

```sql
CREATE UNIQUE INDEX time_hash_idx ON public.transactions (time, hash)
```

## 摄取比特币交易

你已经创建了超表并添加了适当的索引。接下来，摄取一些比特币交易。样本数据文件包含过去五天的比特币交易。这个CSV文件每天更新，所以你总是可以下载最近的比特币交易。将这个数据集插入你的TimescaleDB实例。

<Procedure>

### 摄取比特币交易

1. 下载样本`.csv`文件：<Tag type="download">[tutorial_bitcoin_sample.csv](https://assets.timescale.com/docs/downloads/bitcoin-blockchain/bitcoin_sample.zip)</Tag> 

    ```bash
    wget https://assets.timescale.com/docs/downloads/bitcoin-blockchain/bitcoin_sample.zip 
    ```

1. 解压文件并在需要时更改目录：

    ```bash
    unzip bitcoin_sample.zip
    cd bitcoin_sample
    ```

1. 在`psql`提示符下，将`.csv`文件的内容插入数据库。

    ```bash
    psql -x "postgres://tsdbadmin:<YOUR_PASSWORD_HERE>@<YOUR_HOSTNAME_HERE>:<YOUR_PORT_HERE>/tsdb?sslmode=require"

    \COPY transactions FROM 'tutorial_bitcoin_sample.csv' CSV HEADER;
    ```

    这个过程应该在3-5分钟内完成。

</Procedure>

一旦摄取完成，你的数据库将包含大约150万笔比特币交易。现在，你可以进行你的第一次查询。

## 查询比特币交易

查询最近的五笔非coinbase交易：

```sql
SELECT time, hash, block_id, weight  FROM transactions
WHERE is_coinbase IS NOT TRUE
ORDER BY time DESC
LIMIT 5
```

结果看起来像这样：

<!-- vale Google.Units = NO -->
time               |hash                                                            |block_id|weight|
-------------------|----------------------------------------------------------------|--------|------|
2022-05-30 01:42:17|6543a8e489eade391f099df7066f17783ea2f9d19d644d818ac22bd8fb86005e|  738489|   863|
2022-05-30 01:42:17|a9e2bb3e734e0c0535da4e8ab6e3d0352a44db443d48a861bd5b196575dfd3ff|  738489|   577|
2022-05-30 01:42:17|fd0a9a8c31962107d0a5a0c4ef2a5702e2c9fad6d989e7ac543d87783205a980|  738489|   758|
2022-05-30 01:42:17|e2aedc6026459381485cd57f3e66ea88121e5094c03fa4634193417069058609|  738489|   766|
2022-05-30 01:42:17|429c0d00282645b54bd3c0082800a85d1c952d1764c54dc2a591f97b97c93fbd|  738489|   766|
<!-- vale Google.Units = YES -->

<Highlight type="note">
coinbase交易是每个区块中的第一笔交易。这笔交易包含了矿工的奖励。
</Highlight>

这是另一个示例查询，返回最近的五个区块，包括区块重量、交易计数和美元价值的统计信息：

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
GROUP BY t.block_id
```

结果：

block_id|transaction_count|block_weight|block_value_usd   |
--------|-----------------|------------|------------------|
  738489|             2508|     3991873|402592534.37649953|
  738487|             2231|     3991983| 560811110.1410986|
  738488|             3208|     3991994| 422477674.0440979|
  738486|             2154|     3996197| 481098865.6260999|
  738485|             2602|     3991871| 761258578.3764017|

此时，你的数据库中已经有了比特币区块链数据，并且你已经进行了第一次SQL查询。在下一节中，深入挖掘区块链并使用TimescaleDB超函数生成洞察！

[create_hypertable]: /api/:currentVersion:/hypertable/create_hypertable/


