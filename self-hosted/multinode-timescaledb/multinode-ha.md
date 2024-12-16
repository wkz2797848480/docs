---
标题: 多节点实现高可用性
摘要: 如何配置多节点 TimescaleDB 以实现高可用性
产品: [自托管]
关键词: [多节点，高可用性]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 多节点的高可用性

通过为集群中的每个节点设置一个或多个备用节点，或在块级别本地复制数据，可以使TimescaleDB的多节点安装具有高可用性。

使用备用节点依赖于流复制，您需要类似地[配置单节点HA][single-ha]，尽管配置需要独立应用于每个节点。

要在块级别复制数据，您可以使用TimescaleDB多节点的内置功能，避免复制整个数据节点。访问节点仍然依赖于流复制备用节点，但数据节点不需要额外配置。相反，现有的数据节点池共同负责托管块副本，并处理节点故障。

每种方法都有优点和缺点。为集群中的每个节点设置备用节点确保备用节点在实例级别是相同的，这是提供高可用性的经过验证和测试的方法。然而，它也需要更多的设置和维护镜像集群。

本地复制通常需要更少的资源、节点和配置，并利用内置功能，如添加和删除数据节点，以及每个分布式超表的不同复制因子。然而，只有在数据节点上复制块。

本节的其余部分讨论本地复制。要为每个节点设置备用节点，请按照[单节点HA][single-ha]的说明操作。

## 本地复制

本地复制是一组功能和API，允许您构建高可用的TimescaleDB多节点安装。本地复制的核心是能够将块的副本写入多个数据节点，以便在数据节点故障时有备用的_块副本_。如果一个数据节点失败，它的块应该至少在另一个数据节点上可用。如果一个数据节点永久丢失，可以向集群中添加一个新的数据节点，并且可以从其他数据节点重新复制丢失的块副本以达到所需的块副本数量。

<Highlight type="warning">
TimescaleDB中的本地复制正在开发中，目前缺乏完整的高可用性解决方案的功能。本节描述的某些功能仍然是实验性的。对于生产环境，我们建议为多节点集群中的每个节点设置备用节点。
</Highlight>

### 自动化

类似于单节点PostgreSQL的高可用性配置使用Patroni等系统自动处理故障转移，本地复制需要一个外部实体来协调故障转移、块重新复制和数据节点管理。这种编排不是TimescaleDB默认提供的，因此需要单独实现。下面的部分描述了如何启用本地复制以及实现节点故障时高可用性的步骤。

### 配置本地复制

启用本地复制的第一步是为访问节点配置一个备用节点。这个过程与设置[单节点备用节点][single-ha]相同。

下一步是在分布式超表上启用本地复制。本地复制由`replication_factor`控制，它决定了块复制到多少个数据节点。此设置需要为每个超表单独配置，这意味着同一个数据库可以有一些被复制的分布式超表和其他未被复制的超表。

默认情况下，复制因子设置为`1`，因此没有本地复制。您可以在创建超表时增加这个数字。例如，要在总共三个数据节点上复制数据：

```sql
SELECT create_distributed_hypertable('conditions', 'time', 'location',
 replication_factor => 3);
```

或者，您可以使用[`set_replication_factor`][set_replication_factor]调用来更改现有分布式超表上的复制因子。注意，只有新的块根据更新的复制因子复制。现有的块需要通过将这些块复制到新的数据节点来重新复制（见下文的[节点故障部分](#node-failures)）。

当启用本地复制时，复制发生在您写入数据到表时。在每个`INSERT`和`COPY`调用中，每行数据都被写入多个数据节点。这意味着您不需要执行任何额外的步骤来复制新摄入的数据。当您查询复制的数据时，查询计划器只包括查询计划中每个块的一个副本。

### 节点故障

当一个数据节点失败时，尝试写入失败节点的插入操作会导致错误。这是为了在数据节点再次可用时保持数据一致性。您可以使用[`alter_data_node`][alter_data_node]调用来标记失败的数据节点为不可用，运行此查询：

```sql
SELECT alter_data_node('data_node_2', available => false);
```

设置`available => false`意味着数据节点不再用于读写查询。

要故障转移读取，[`alter_data_node`][alter_data_node]调用找到所有不可用数据节点是主查询目标的块，并故障转移到另一个数据节点上的块副本。然而，如果有些块没有副本可以故障转移，会发出警告。对于没有块副本在任何其他数据节点上的块，读取继续失败。

要故障转移写入，任何打算写入失败节点的活动都会通过更改访问节点上的元数据，将涉及的块标记为特定失败节点的陈旧。这只对本地复制的块进行。这允许您在失败的节点被标记为不可用时，继续在其他数据节点上的其他块副本上写入。对于没有块副本在任何其他数据节点上的块，写入继续失败。同样请注意，失败节点上没有写入的块不受影响。

当您标记一个块为陈旧时，该块变得复制不足。当失败的数据节点变得可用时，可以使用[`copy_chunk`][copy_chunk] API重新平衡这样的块。

如果等待数据节点恢复不是一个选项，无论是因为时间过长还是节点永久失败，可以删除它。要能够删除数据节点，其所有块都必须在其他数据节点上至少有一个副本。例如：

```sql
SELECT delete_data_node('data_node_2', force => true);
WARNING:  distributed hypertable "conditions" is under-replicated
```

当您删除数据节点意味着集群不再达到所需的复制因子时，使用`force`选项。这将是正常的情况，除非数据节点没有块或分布式超表的块副本多于配置的复制因子。

<Highlight type="important">
如果强制删除数据节点意味着多节点集群永久丢失数据，您不能强制删除数据节点。
</Highlight>

当您成功移除失败的数据节点，或标记失败的数据节点为不可用时，一些数据块可能缺少副本，但查询和插入再次正常工作。然而，直到所有块都被完全复制，集群仍处于脆弱状态。

当您恢复失败的数据节点或再次标记它为可用时，您可以使用此查询查看需要复制的块：

```sql
SELECT chunk_schema, chunk_name, replica_nodes, non_replica_nodes
FROM timescaledb_experimental.chunk_replication_status
WHERE hypertable_name = 'conditions' AND num_replicas < desired_num_replicas;
```

此查询的输出如下所示：

```sql
     chunk_schema      |      chunk_name       | replica_nodes |     non_replica_nodes
-----------------------+-----------------------+---------------+---------------------------
 _timescaledb_internal | _dist_hyper_1_1_chunk | {data_node_3} | {data_node_1,data_node_2}
 _timescaledb_internal | _dist_hyper_1_3_chunk | {data_node_1} | {data_node_2,data_node_3}
 _timescaledb_internal | _dist_hyper_1_4_chunk | {data_node_3} | {data_node_1,data_node_2}
(3 rows)
```

有了块复制状态视图的信息，可以将复制不足的块复制到新节点，以确保块具有足够数量的副本。例如：

```sql
CALL timescaledb_experimental.copy_chunk('_timescaledb_internal._dist_hyper_1_1_chunk', 'data_node_3', 'data_node_2');
```

<Highlight type="important">
当您恢复块复制时，操作使用超过一个事务。这意味着它不能自动回滚。如果您在操作完成之前取消操作，将记录复制的操作ID。您可以使用此操作ID来清理取消操作留下的任何状态。例如：

```sql
CALL timescaledb_experimental.cleanup_copy_chunk_operation('ts_copy_1_31');
```

</Highlight>

[set_replication_factor]:  /api/:currentVersion:/distributed-hypertables/set_replication_factor
[single-ha]: /self-hosted/:currentVersion:/replication-and-ha/
[alter_data_node]: /api/:currentVersion:/distributed-hypertables/alter_data_node/
[copy_chunk]:/api/:currentVersion:/distributed-hypertables/copy_chunk_experimental

