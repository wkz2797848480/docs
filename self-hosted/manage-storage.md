---
标题: 使用表空间管理存储
摘要: 通过在表空间之间移动数据块来节省数据存储空间。
产品: [自托管]
关键词: [存储，表空间]
标签: [移动，管理，数据块]
---

import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";

# 通过表空间管理存储

如果您在自有硬件上运行Timescale，您可以通过在表空间之间移动数据块来节省存储空间。通过将旧数据块移动到更便宜、更慢的存储上，您可以在仍然使用更快、更昂贵的存储来频繁访问数据的同时节省存储成本。移动不常访问的数据块还可以提高性能，因为它将历史数据与更近期数据的持续读写工作负载隔离开来。

<Highlight type="note">
使用表空间是管理Timescale数据存储成本的一种方式。您还可以使用[压缩](/use-timescale/latest/compression)和[数据保留](/use-timescale/latest/data-retention)来减少您的存储需求。
</Highlight>

<ConsiderCloud />

## 移动数据

要将数据块移动到新的表空间，您首先需要创建新的表空间并设置存储挂载点。然后，您可以使用[`move_chunk`][api-move-chunk] API调用来将单个数据块从默认表空间移动到新的表空间。`move_chunk`命令还允许您将属于这些数据块的索引移动到适当的表空间。

此外，`move_chunk`允许在迁移过程中重新排序数据块。这可以用来加速您的查询，并且与[`reorder_chunk`命令][api-reorder-chunk]的工作方式类似。

<Highlight type="note">
您必须以超级用户（例如`postgres`用户）身份登录才能使用`move_chunk()` API调用。
</Highlight>

<Procedure>

### 移动数据

1.  创建一个新的表空间。在这个例子中，表空间被称为`history`，它由`postgres`超级用户拥有，挂载点是`/mnt/history`：

    ```sql
    CREATE TABLESPACE history
    OWNER postgres
    LOCATION '/mnt/history';
    ```

1.  列出您想要移动的数据块。在这个例子中，包含超过两天旧数据的数据块：

    ```sql
    SELECT show_chunks('conditions', older_than => INTERVAL '2 days');
    ```

1.  将数据块及其索引移动到新的表空间。您还可以在这一步重新排序数据。在这个例子中，名为`_timescaledb_internal._hyper_1_4_chunk`的数据块被移动到`history`表空间，并根据其时间索引重新排序：

    ```sql
    SELECT move_chunk(
      chunk => '_timescaledb_internal._hyper_1_4_chunk',
      destination_tablespace => 'history',
      index_destination_tablespace => 'history',
      reorder_index => '_timescaledb_internal._hyper_1_4_chunk_netdata_time_idx',
      verbose => TRUE
    );
    ```

1.  您可以通过查询`pg_tables`来验证数据块现在是否位于正确的表空间：

    ```sql
    SELECT tablename from pg_tables
      WHERE tablespace = 'history' and tablename like '_hyper_%_%_chunk';
    ```

    您还可以验证索引是否位于正确的位置：

    ```sql
    SELECT indexname FROM pg_indexes WHERE tablespace = 'history';
    ```

</Procedure>

## 批量移动数据

要一次性移动多个数据块，使用`FROM show_chunks(...)`选择您想要移动的数据块。例如，要移动包含1到3周旧数据的数据块，在名为`example`的超表中：

```sql
SELECT move_chunk(
  chunk => i,
  destination_tablespace => '<TABLESPACE>')
FROM show_chunks('example', now() - INTERVAL '1 week', now() - INTERVAL '3 weeks') i;
```

## 示例

在将数据块移动到较慢的表空间后，您可以将其移回默认的、更快的表空间：

```sql
SELECT move_chunk(
  chunk => '_timescaledb_internal._hyper_1_4_chunk',
  destination_tablespace => 'pg_default',
  index_destination_tablespace => 'pg_default',
  reorder_index => '_timescaledb_internal._hyper_1_4_chunk_netdata_time_idx'
);
```

您可以将数据块移动到较慢的表空间，但将该数据块的索引保留在默认的、更快的表空间：

```sql
SELECT move_chunk(
  chunk => '_timescaledb_internal._hyper_1_4_chunk',
  destination_tablespace => 'history',
  index_destination_tablespace => 'pg_default',
  reorder_index => '_timescaledb_internal._hyper_1_4_chunk_netdata_time_idx'
);
```

您还可以将数据保留在`pg_default`，但将索引移动到`history`。或者，您可以设置一个名为`history_indexes`的第三个表空间，并将数据移动到`history`，将索引移动到`history_indexes`。

在Timescale 2.0及更高版本中，您可以使用作业调度框架中的`move_chunk`。更多信息，请参见[用户定义操作部分][actions]。

[actions]: /use-timescale/:currentVersion:/user-defined-actions/
[api-move-chunk]: /api/:currentVersion:/hypertable/move_chunk
[api-reorder-chunk]: /api/:currentVersion:/hypertable/reorder_chunk

