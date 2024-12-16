---
标题: 创建分布式超表
摘要: 在多节点 Timescale 实例中创建分布式超表
产品: [自托管]
关键词: [分布式超表，多节点，创建]
---

import 多节点已弃用 from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 创建分布式超表

如果您有一个[多节点环境][multi-node]，您可以在数据节点之间创建一个分布式超表。首先创建一个标准的PostgreSQL表，然后将其转换为分布式超表。

<Highlight type="important">
在创建分布式超表之前，您需要先设置好您的多节点集群。关于如何设置多节点，请参阅[多节点部分](/self-hosted/latest/multinode-timescaledb/)。
</Highlight>

<Procedure>

### 创建分布式超表

1.  在您的多节点集群的接入节点上，创建一个标准的[PostgreSQL表][postgres-createtable]：

    ```sql
    CREATE TABLE conditions (
      time        TIMESTAMPTZ       NOT NULL,
      location    TEXT              NOT NULL,
      temperature DOUBLE PRECISION  NULL,
      humidity    DOUBLE PRECISION  NULL
    );
    ```

1.  将表转换为分布式超表。指定您想要转换的表的名称，持有其时间值的列，以及一个空间分区参数。

    ```sql
    SELECT create_distributed_hypertable('conditions', 'time', 'location');
    ```

</Procedure>

[multi-node]: /self-hosted/:currentVersion:/multinode-timescaledb/
[postgres-createtable]: https://www.postgresql.org/docs/current/sql-createtable.html
