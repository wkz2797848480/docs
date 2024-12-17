---
标题: 能源消耗数据教程 —— 设置压缩
摘要: 压缩数据集，以便能更高效地存储能源消耗数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，查询]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析能源消耗数据
---
# 设置压缩并压缩数据集

您现在已经看到了如何为您的能耗数据集创建一个超表并查询它。当摄入这样的数据集时，很少需要更新旧数据，随着时间的推移，表中的数据量会增长。随着时间的推移，您最终会有很多数据，由于这些数据大多是不可变的，您可以压缩它以节省空间并避免产生额外成本。

可以使用像 ZFS 和 Btrfs 提供的面向磁盘的压缩，但由于 TimescaleDB 是为处理事件导向数据（例如时间序列）而构建的，因此它支持在超表中压缩数据。

TimescaleDB 压缩允许您以更高效的格式存储数据，与普通的 PostgreSQL 表相比，压缩比率可高达 20 倍，但这当然高度依赖于数据和配置。

TimescaleDB 压缩是在 PostgreSQL 中本地实现的，不需要特殊的存储格式。相反，它依赖于 PostgreSQL 的特性，在压缩之前将数据转换为列式格式。使用列式格式可以更好地压缩比率，因为相似的数据被存储在相邻位置。有关压缩格式的更多详细信息，您可以查看[压缩设计][compression-design]部分。

压缩数据的一个有益副作用是某些查询会显著加快，因为需要读取到内存中的数据减少了。

<Procedure>

## 压缩设置

1. 使用例如 `psql` 连接到包含能耗数据集的 Timescale 数据库。
1. 使用 `ALTER TABLE` 命令在表上启用压缩，并选择适当的 segment-by 和 order-by 列：

    ```sql
    ALTER TABLE metrics 
    SET (
        timescaledb.compress, 
        timescaledb.compress_segmentby='type_id', 
        timescaledb.compress_orderby='created DESC'
    );
    ``` 
    根据您的 segment-by 和 order-by 列的选择，您可以获得非常不同的性能和压缩比率。要了解更多关于如何选择正确的列，请查看[这里][segment-by-columns]。
1. 您可以手动使用以下方式压缩超表的所有块：

    ```sql
    SELECT compress_chunk(c) from show_chunks('metrics') c;
    ```
    您还可以[自动压缩][automatic-compression]，通过添加[压缩策略][add_compression_policy]，稍后将介绍。

1. 现在您已经压缩了表，您可以比较压缩前后数据集的大小：

    ```sql
    SELECT 
        pg_size_pretty(before_compression_total_bytes) as before,
        pg_size_pretty(after_compression_total_bytes) as after
     FROM hypertable_compression_stats('metrics');
    ```
    这显示了数据使用量的显著改进：

    ```sql
     before | after 
    --------+-------
     180 MB | 16 MB
    (1 row)
    ```

</Procedure>

## 添加压缩策略

为了避免每次有数据需要压缩时都运行压缩步骤，您可以设置一个压缩策略。压缩策略允许您压缩超过特定年龄的数据，例如，压缩所有超过 8 天的块：

```sql
SELECT add_compression_policy('metrics', INTERVAL '8 days');
```

压缩策略按照定期计划运行，默认情况下每天一次，这意味着使用上述设置，您可能有多达 9 天的未压缩数据。

您可以在[add_compression_policy][add_compression_policy]部分找到有关压缩策略的更多信息。

## 利用查询加速

之前，压缩被设置为按 `type_id` 列值分割。这意味着通过过滤或在该列上分组获取数据将更加高效。排序也设置为 `created` 降序，因此如果您运行尝试以该排序顺序排序数据的查询，您应该看到性能优势。

例如，如果您运行上一节的查询示例：
```sql
SELECT time_bucket('1 day', created, 'Europe/Berlin') AS "time",
        round((last(value, created) - first(value, created)) * 
100.) / 100. AS value
FROM metrics                                     
WHERE type_id = 5
GROUP BY 1;
```

当数据集被压缩和未被压缩时，您应该看到相当大的性能差异。自己尝试一下，运行之前的查询，解压数据集，再次运行它，并计时执行时间。您可以通过运行以下命令在 psql 中启用计时查询时间：

```sql
    \timing
```

要解压整个数据集，请运行：
```sql
    SELECT decompress_chunk(c) from show_chunks('metrics') c;
```

在一个示例设置中，观察到的性能提升是一个数量级，压缩时为 30 毫秒，未压缩时为 360 毫秒。

自己尝试一下，看看您得到的结果！

[segment-by-columns]: /use-timescale/:currentVersion:/compression/about-compression/#segment-by-columns
[automatic-compression]: /tutorials/:currentVersion:/energy-data/compress-energy/#add-a-compression-policy
[compression-design]: /use-timescale/:currentVersion:/compression/compression-design/
[add_compression_policy]: /api/:currentVersion:/compression/add_compression_policy/
