---
标题: 查询比特币区块链 —— 设置压缩
摘要: 压缩数据集，以便更高效地存储比特币区块链。
产品: [云服务]
关键词: [初学者，加密货币，区块链，比特币，金融，分析]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 查询比特币区块链
---

# 设置压缩并压缩数据集

您现在已经知道如何为您的比特币数据集创建超表并查询区块链数据。在摄入类似这样的数据集时，很少需要更新旧数据，随着时间的推移，表中的数据量会增加。随着时间的推移，您最终会积累大量数据，由于这些数据大多是不可变的，您可以压缩它们以节省空间并避免产生额外成本。

虽然可以使用像ZFS和Btrfs这样的磁盘导向压缩，但由于TimescaleDB是为了处理事件导向数据（如时间序列）而构建的，因此它提供了超表中压缩数据的支持。

TimescaleDB压缩允许您以一种效率更高的格式存储数据，与普通的PostgreSQL表相比，可以实现高达20倍的压缩比率，但这当然高度依赖于数据和配置。

TimescaleDB压缩在PostgreSQL中本地实现，不需要特殊的存储格式。相反，它依赖于PostgreSQL的特性，在压缩之前将数据转换为列式格式。使用列式格式可以更好地压缩比率，因为相似的数据被存储在相邻位置。有关压缩格式的更多详细信息，请参阅[压缩设计][compression-design]部分。

压缩数据的一个有益副作用是，某些查询速度显著加快，因为需要读取到内存中的数据减少了。

<Procedure>

## 压缩设置

在前面的部分中，您学习了如何在`transactions`表上执行不同的查询，因此如果您想要压缩该表的数据，请按照以下步骤操作：

1.  使用`psql`连接到包含比特币数据集的Timescale数据库。
2.  使用`ALTER TABLE`命令启用表的压缩，并选择适当的`segment-by`和`order-by`列：

    ```sql
    ALTER TABLE transactions 
    SET (
        timescaledb.compress, 
        timescaledb.compress_segmentby='block_id', 
        timescaledb.compress_orderby='time DESC'
    );
    ``` 

    根据您选择的`segment-by`和`order-by`列，您可能会得到非常不同的性能和压缩比率。要了解更多关于如何选择正确的列，请参阅[此处][segment-by-columns]。

3.  您可以手动使用以下方式压缩超表的所有块：

    ```sql
    SELECT compress_chunk(c) from show_chunks('transactions') c;
    ```

    您还可以[自动压缩][automatic-compression]，通过添加[压缩策略][add_compression_policy]，稍后将介绍。

4.  现在您已经压缩了表，可以比较压缩前后数据集的大小：

    ```sql
    SELECT 
        pg_size_pretty(before_compression_total_bytes) as before,
        pg_size_pretty(after_compression_total_bytes) as after
     FROM hypertable_compression_stats('transactions');
    ```

    这显示了数据使用量的显著改进：

    ```sql
     before  | after  
    ---------+--------
    1307 MB | 237 MB   
    (1 row)
    ```

</Procedure>

## 添加压缩策略

为了避免每次都运行压缩步骤，您可以设置压缩策略。压缩策略允许您压缩超过特定年龄的数据，例如，压缩所有超过8天的数据块：

```sql
SELECT add_compression_policy('transactions', INTERVAL '8 days');
```

压缩策略按定期计划运行，默认情况下每天一次，这意味着您可能有多达9天的未压缩数据。

您可以在[add_compression_policy][add_compression_policy]部分找到更多关于压缩策略的信息。

## 利用查询速度提升

之前，压缩被设置为按`block_id`列值分段。这意味着通过过滤或分组该列来获取数据将更加高效。排序也设置为按时间降序，因此如果您运行尝试以该排序顺序排列数据的查询，您应该能看到性能提升。

例如，如果您运行上一节的查询示例：
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

当数据集被压缩时，您应该能看到与未压缩时相比的显著性能差异。自己尝试一下，先运行之前的查询，然后解压数据集，再次运行它，并计时执行时间。您可以通过运行以下命令在psql中启用查询时间计时：

```sql
    \timing
```

要解压整个数据集，请运行：
```sql
    SELECT decompress_chunk(c) from show_chunks('transactions') c;
```

在一个示例设置中，观察到的性能提升是两个数量级，压缩时为15毫秒，未压缩时为1秒。

自己尝试一下，看看您能得到什么结果！

[segment-by-columns]: /use-timescale/:currentVersion:/compression/about-compression/#segment-by-columns
[automatic-compression]: /tutorials/:currentVersion:/blockchain-query/blockchain-compress/#add-a-compression-policy
[compression-design]: /use-timescale/:currentVersion:/compression/compression-design/
[add_compression_policy]: /api/:currentVersion:/compression/add_compression_policy/

