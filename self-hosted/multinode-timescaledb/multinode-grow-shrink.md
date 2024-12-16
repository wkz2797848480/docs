---
标题: 扩展和收缩多节点
摘要: 在多节点集群中添加和移除数据节点
产品: [自托管]
关键词: [多节点，数据节点]
标签: [添加，移除]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 多节点扩展与缩减

在多节点环境中工作时，您可能会发现随着时间的推移，您的集群需要更多或更少的数据节点。在选择创建分布式超表时，您可以选择使用多少个可用节点。您还可以添加和移除集群中的数据节点，并根据需要在数据节点之间移动数据块以释放存储空间。

## 查看哪些数据节点正在使用

您可以使用此查询检查哪些数据节点被分布式超表使用。在这个例子中，我们的分布式超表称为`conditions`：

```sql
SELECT hypertable_name, data_nodes
FROM timescaledb_information.hypertables
WHERE hypertable_name = 'conditions';
```

此查询的结果如下所示：

```sql
hypertable_name |              data_nodes
-----------------+---------------------------------------
conditions      | {data_node_1,data_node_2,data_node_3}
```

## 选择分布式超表使用多少节点

默认情况下，当您创建一个分布式超表时，它会使用所有可用的数据节点。要将其限制在特定节点上，请在[`create_distributed_hypertable`][create_distributed_hypertable]时传递`data_nodes`参数。

## 添加新数据节点

当您向数据库添加额外的数据节点时，需要将它们添加到分布式超表中，以便数据库可以使用它们。

<Procedure>

### 将新数据节点添加到分布式超表

1.  在访问节点上，在`psql`提示符下，添加数据节点：

    ```sql
    SELECT add_data_node('node3', host => 'dn3.example.com');
    ```

1.  将新数据节点附加到分布式超表：

    ```sql
    SELECT attach_data_node('node3', hypertable => 'hypertable_name');
    ```

<Highlight type="important">
当您附加新数据节点时，分布式超表的分区配置会更新以考虑额外的数据节点，并且散列分区的数量会自动增加以匹配。您可以通过将函数参数`repartition`设置为`FALSE`来阻止这种情况发生。
</Highlight>

</Procedure>

## 在数据块之间移动数据 <Tag type="experimental">实验性</Tag>

当您将新数据节点附加到分布式超表时，您可以将超表中的现有数据移动到新节点，以释放现有节点上的存储空间，并更好地利用增加的容量。

<Highlight type="warning">
在数据节点之间移动数据块的能力是一个正在积极开发中的实验性功能。我们建议您不要在生产环境中使用此功能。
</Highlight>

使用此查询移动数据：

```sql
CALL timescaledb_experimental.move_chunk('_timescaledb_internal._dist_hyper_1_1_chunk', 'data_node_3', 'data_node_2');
```

移动操作使用多个事务，这意味着如果出现问题，您不能自动回滚事务。如果移动操作失败，失败会记录一个操作ID，您可以使用它来清理涉及节点上留下的任何状态。

使用此查询在移动失败后进行清理。在这个例子中，失败移动的操作ID是`ts_copy_1_31`：

```sql
CALL timescaledb_experimental.cleanup_copy_chunk_operation('ts_copy_1_31');
```

## 移除数据节点

您也可以从现有的分布式超表中移除数据节点。

<Highlight type="warning">
您不能移除仍包含分布式超表数据的数据节点。在移除数据节点之前，请检查它是否已删除或移动了所有数据，或者您是否已将数据复制到其他数据节点上。
</Highlight>

使用此查询移除数据节点。在这个例子中，我们的分布式超表称为`conditions`：

```sql
SELECT detach_data_node('node1', hypertable => 'conditions');
```

[create_distributed_hypertable]: /api/:currentVersion:/distributed-hypertables/create_distributed_hypertable/

