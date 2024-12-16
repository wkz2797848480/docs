---
标题: 多节点配置
摘要: 配置一个多节点 TimescaleDB 实例
产品: [自托管]
关键词: [配置，设置，多节点]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 多节点配置

除了[常规的TimescaleDB配置][timescaledb-configuration]之外，建议您还配置特定于多节点操作的额外设置。

## 更新设置

这些设置可以在各个节点的`postgresql.conf`文件中配置。`postgresql.conf`文件通常位于`data`目录中，但您可以通过连接到节点并使用`psql`执行以下命令来找到正确的路径：

```sql
SHOW config_file;
```

修改`postgresql.conf`文件后，重新加载配置以查看您的更改：

```bash
pg_ctl reload
```

### `max_prepared_transactions`

如果尚未设置，请确保所有数据节点上的`max_prepared_transactions`设置为非零值，起始点设为`150`。

### `enable_partitionwise_aggregate`

在访问节点上，将`enable_partitionwise_aggregate`参数设置为`on`。这确保查询被推送到数据节点，并提高查询性能。

### `jit`

在访问节点上，将`jit`设置为`off`。目前，JIT与分布式查询的兼容性不佳。但是，您可以成功在数据节点上启用JIT。

### `statement_timeout`

在数据节点上，禁用`statement_timeout`。如果需要启用此设置，请仅在访问节点上启用和配置。该设置在PostgreSQL中默认禁用，但在特定环境中可能有用。

### `wal_level`

在数据节点上，将`wal_level`设置为`logical`或更高，以便[移动][move_chunk]或[复制][copy_chunk]数据节点之间的数据块。如果同时移动许多数据块，考虑增加`max_wal_senders`和`max_replication_slots`。

### 事务隔离级别

为了保持一致性，如果事务隔离级别设置为`READ COMMITTED`，每当发生分布式操作时，它将自动升级为`REPEATABLE READ`。如果隔离级别是`SERIALIZABLE`，则不改变。

[copy_chunk]: /api/:currentVersion:/distributed-hypertables/copy_chunk_experimental
[move_chunk]: /api/:currentVersion:/distributed-hypertables/move_chunk_experimental
[timescaledb-configuration]: /self-hosted/:currentVersion:/configuration/
