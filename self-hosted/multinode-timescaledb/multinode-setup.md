---
标题: 在自托管的 TimescaleDB 上搭建多节点
摘要: 如何搭建自托管的多节点实例
产品: [自托管]
关键词: [多节点，自托管]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 在自托管的TimescaleDB上设置多节点

要在自托管的TimescaleDB实例上设置多节点，您需要：

*   一个作为访问节点（AN）的PostgreSQL实例
*   一个或多个作为数据节点（DN）的PostgreSQL实例
*   所有节点上[安装][install]并[设置][setup]了TimescaleDB
*   所有节点上的超级用户角色访问权限，例如`postgres`

访问节点和数据节点必须作为单独的TimescaleDB实例开始。它们应该是具有运行中的PostgreSQL服务器和加载的TimescaleDB扩展的主机。有关安装自托管TimescaleDB实例的更多信息，请参见[安装说明][install]。此外，您还可以配置[多节点的高可用性][multi-node-ha]以增加冗余和弹性。

多节点TimescaleDB架构由存储分布式超表元数据并执行集群内查询规划的访问节点（AN）和存储分布式超表数据子集并本地执行查询的数据节点（DN）组成。有关多节点架构的更多信息，请参见[关于多节点][about-multi-node]。

如果您打算在多节点环境中使用连续聚合，请查看[连续聚合][caggs]部分的额外考虑事项。

## 在自托管的TimescaleDB上设置多节点

当您在访问节点上安装了TimescaleDB，并根据您的需要设置了多个数据节点后，您可以设置多节点并创建分布式超表。

<Highlight type="note">
在开始之前，请确保您已经考虑了要为您的多节点集群使用哪种分区方法。有关多节点和架构的更多信息，请参见[关于多节点部分](/self-hosted/latest/multinode-timescaledb/about-multinode/)。
</Highlight>

<Procedure>

### 在自托管的TimescaleDB上设置多节点

1.  在访问节点（AN）上，运行此命令并提供您想要添加的第一个数据节点（DN1）的主机名：

    ```sql
    SELECT add_data_node('dn1', 'dn1.example.com')
    ```

1.  对所有其他数据节点重复此操作：

    ```sql
    SELECT add_data_node('dn2', 'dn2.example.com')
    SELECT add_data_node('dn3', 'dn3.example.com')
    ```

1.  在访问节点上，使用您选择的分区方法创建分布式超表。在这个例子中，分布式超表称为`example`，它根据`time`和`location`进行分区：

    ```sql
    SELECT create_distributed_hypertable('example', 'time', 'location');
    ```

1.  向超表中插入一些数据。例如：

    ```sql
    INSERT INTO example VALUES ('2020-12-14 13:45', 1, '1.2.3.4');
    ```

</Procedure>

当您设置好多节点安装后，您可以配置您的集群。有关更多信息，请参见[配置部分][configuration]。

[about-multi-node]: /self-hosted/:currentVersion:/multinode-timescaledb/about-multinode/
[caggs]: /use-timescale/:currentVersion:/continuous-aggregates/about-continuous-aggregates/#using-continuous-aggregates-in-a-multi-node-environment
[configuration]: /self-hosted/:currentVersion:/multinode-timescaledb/multinode-config/
[install]: /self-hosted/latest/install/
[multi-node-ha]: /self-hosted/:currentVersion:/multinode-timescaledb/multinode-ha/
[setup]: /self-hosted/:currentVersion:/install/

