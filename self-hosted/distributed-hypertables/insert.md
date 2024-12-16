---
标题: 插入数据
摘要: 如何将数据插入分布式超表中
产品: [自托管]
关键词: [写入，分布式超表]
标签: [摄取，插入]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 插入数据

您可以使用 `INSERT` 语句将数据插入分布式超表。语法与标准超表或 PostgreSQL 表相同。例如：

```sql
INSERT INTO conditions(time, location, temperature, humidity)
  VALUES (NOW(), 'office', 70.0, 50.0);
```

## 优化数据插入

分布式超表比标准超表有更高的网络负载，因为它们必须将从接入节点到数据节点的插入操作进行推送。您可以优化插入模式以减少负载。

### 批量插入数据

通过批量处理 `INSERT` 语句来减少负载，而不是将每个插入操作作为单独的事务执行。

接入节点首先通过确定每行数据应属于哪个数据节点，将批量数据分割成更小的批次。然后，它将每个批次写入正确的数据节点。

### 优化插入批次大小

当向分布式超表插入数据时，接入节点尝试将 `INSERT` 语句转换为更高效的 [`COPY`][postgresql-copy] 操作。但如果：

*   `INSERT` 语句包含 `RETURNING` 子句，并且
*   超表具有可能更改返回数据的触发器

在这种情况下，计划器使用多行预处理语句将数据插入每个数据节点。它将原始插入语句分割成这些子语句。您可以通过运行 [`EXPLAIN`][postgresql-explain] 来查看您的 `INSERT` 语句的计划。

在预处理语句中，接入节点可以在将数据刷新到数据节点之前缓冲一定数量的行。默认情况下，这个数字是 1000。您可以通过更改 `timescaledb.max_insert_batch_size` 设置来优化这一点，例如减少必须发送的独立批次的数量。

最大批次大小有一个上限。这等于预处理语句允许的最大参数数量，目前是 32,767 个参数，除以每行的列数。例如，如果您有一个具有 10 列的分布式超表，您可以设置的最高批次大小是 3276。

有关更改 `timescaledb.max_insert_batch_size` 的更多信息，请参见 [配置][config] 部分。

### 使用复制语句代替

[`COPY`][postgresql-copy] 可能比分布式超表上的 `INSERT` 执行得更好。但它不支持某些功能，例如使用 `ON CONFLICT` 子句进行冲突处理。

要从文件复制到您的超表，请运行：

```sql
COPY <HYPERTABLE> FROM '<FILE_PATH>';
```

执行 [`COPY`][postgresql-copy] 时，接入节点将每个数据节点切换到复制模式。然后，它将每行数据流式传输到正确的数据节点。

[config]: /self-hosted/:currentVersion:/configuration/timescaledb-config/#distributed-hypertables
[postgresql-copy]: https://www.postgresql.org/docs/14/sql-copy.html 
[postgresql-explain]: https://www.postgresql.org/docs/14/sql-explain.html
