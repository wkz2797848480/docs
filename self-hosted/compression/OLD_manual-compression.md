---
标题: 手动压缩
摘要: 手动压缩超表
产品: [自托管]
关键词: [压缩，超表]
---


# 手动压缩块

在大多数情况下，自动压缩策略就足够了。但是，如果您想要更多地控制压缩，您也可以手动压缩特定的块。

## 手动压缩块

开始之前，您需要一个要压缩的块列表。在这个例子中，您使用一个名为`example`的超表，并压缩三天以前的块。

<Highlight type="warning">
压缩会改变您磁盘上的数据，因此在开始之前请务必备份。
</Highlight>

<Procedure>

### 选择要压缩的块

1. 在psql提示符下，选择表`example`中所有三天以前的块：

    ```sql
    SELECT show_chunks('example', older_than => INTERVAL '3 days');
    ```

1. 这会返回一个块列表。记下块的名称：

    ```sql
    ||show_chunks|
    |---|---|
    |1|_timescaledb_internal_hyper_1_2_chunk|
    |2|_timescaledb_internal_hyper_1_3_chunk|
    ```

</Procedure>

当您对块列表满意后，您可以使用块名称手动压缩每个块。

<Procedure>

### 手动压缩块

1. 在psql提示符下，压缩块：

    ```sql
    SELECT compress_chunk( '<chunk_name>');
    ```

1. 使用此命令检查压缩结果：

    ```sql
    SELECT *
    FROM chunk_compression_stats('example');
    ```

    结果显示给定超表的块、它们的压缩状态和其他一些统计信息：

    ```sql
    |chunk_schema|chunk_name|compression_status|before_compression_table_bytes|before_compression_index_bytes|before_compression_toast_bytes|before_compression_total_bytes|after_compression_table_bytes|after_compression_index_bytes|after_compression_toast_bytes|after_compression_total_bytes|node_name|
    |---|---|---|---|---|---|---|---|---|---|---|
    |_timescaledb_internal|_hyper_1_1_chunk|Compressed|8192 bytes|16 kB|8192 bytes|32 kB|8192 bytes|16 kB|8192 bytes|32 kB||
    |_timescaledb_internal|_hyper_1_20_chunk|Uncompressed||||||||||
    ```

1. 重复此过程以压缩所有您想要压缩的块。

</Procedure>

## 单条命令手动压缩块

或者，您可以选择块并使用单条命令压缩它们，方法是使用`show_chunks`命令的输出来压缩每个块。例如，使用此命令压缩一周到三周之间的块（如果它们尚未被压缩）：

```sql
SELECT compress_chunk(i, if_not_compressed => true)
    FROM show_chunks(
        'example',
        now()::timestamp - INTERVAL '1 week',
        now()::timestamp - INTERVAL '3 weeks'
    ) i;
```

<Highlight type="note">
如果您已经使用`compress_chunk_time_interval`设置了压缩策略，然后您在不指定特定参数的情况下使用`compress_chunk`进行手动压缩，手动压缩将遵循之前在策略中设置的时间间隔。
</Highlight>

## 压缩时归档未压缩的块

在Timescale 2.9及更高版本中，您可以在压缩过程中将多个未压缩的块归档到之前压缩过的块中。这允许您拥有更小的未压缩块间隔，从而减少未压缩数据使用的磁盘空间。例如，如果您的数据中有多个较小的未压缩块，您可以将它们归档到一个单一的压缩块中。

要将未压缩的块归档到压缩块中，请修改压缩设置以设置压缩块时间间隔，并运行压缩操作以在压缩时归档块。

```sql
ALTER TABLE example SET (timescaledb.compress_chunk_time_interval = '<time_interval>');
SELECT compress_chunk(c, if_not_compressed => true)
    FROM show_chunks(
        'example',
        now()::timestamp - INTERVAL '1 week',
    ) c;
```

您选择的时间间隔必须是未压缩块间隔的倍数。例如，如果您的未压缩块间隔是一周，您的压缩块的`<time_interval>`可以是两周或六周，但不是一个月。

