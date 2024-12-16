---
标题: TimescaleDB 配置与调优
摘要: 如何更改 TimescaleDB 的配置设置
产品: [自托管]
关键词: [配置，设置]
标签: [调优]
---

import 多节点已经弃用 from "versionContent/_partials/_multi-node-deprecation.mdx";

# TimescaleDB配置和调优

就像您可以在PostgreSQL中调整设置一样，TimescaleDB也提供了许多配置设置，这些设置可能对您的特定安装和性能需求有用。这些设置也可以在`postgresql.conf`文件中设置，或者作为启动PostgreSQL时的命令行参数。

## 策略

### `timescaledb.max_background_workers (int)`

分配给TimescaleDB的最大后台工作进程数。设置为至少1加上在PostgreSQL实例中加载了TimescaleDB扩展的数据库数量。默认值为16。

## 查询规划与执行

### `timescaledb.enable_chunkwise_aggregation (bool)`
如果启用，在查询规划期间聚合转换为部分聚合。聚合的第一部分在每个块的基础上执行。然后，这些部分结果被组合并完成。分割聚合可以减小创建的哈希表大小，并增加数据局部性，从而加快查询速度。

### `timescaledb.vectorized_aggregation (bool)`
启用或禁用查询执行器中的向量化优化。例如，压缩块上的`sum()`聚合函数可以通过这种方式进行优化。

### `timescaledb.enable_merge_on_cagg_refresh  (bool)`
设置为`TRUE`可以在连续聚合存在少量变化时显著减少写入数据量，减少刷新[连续聚合][continuous-aggregates]的I/O成本，并生成较少的预写日志（WAL）。

## 分布式超级表

<MultiNodeDeprecation />

### `timescaledb.enable_2pc (bool)`
为分布式超级表启用两阶段提交。如果禁用，它将改用一阶段提交，这更快但可能导致数据不一致。默认启用。

### `timescaledb.enable_per_data_node_queries`
如果启用，TimescaleDB将同一超级表所属的不同块合并为每个数据节点的单个查询。默认启用。

### `timescaledb.max_insert_batch_size (int)`
当作为访问节点时，TimescaleDB将插入的元组批次跨多个数据节点分割。它每个数据节点批次多达`max_insert_batch_size`元组后才刷新。设置为0禁用批次处理，恢复为逐个元组插入。默认值为1000。

### `timescaledb.enable_connection_binary_data (bool)`
启用集群节点间交换数据的二进制格式。默认启用。

### `timescaledb.enable_client_ddl_on_data_nodes (bool)`
启用客户端在数据节点上执行DDL操作，并不仅限于访问节点执行DDL操作。默认禁用。

### `timescaledb.enable_async_append (bool)`
启用优化，跨数据节点异步运行远程查询。默认启用。

### `timescaledb.enable_remote_explain (bool)`
启用从远程节点获取和显示`EXPLAIN`输出。这需要将查询发送到数据节点，因此可能受到网络连接和数据节点可用性的影响。默认禁用。

### `timescaledb.remote_data_fetcher (enum)`
根据您计划运行的查询类型选择数据提取器类型，可以是`copy`、`cursor`或`auto`。默认为`auto`。

### `timescaledb.ssl_dir (string)`
指定连接到使用证书认证的数据节点时搜索用户证书和密钥的路径。默认为PostgreSQL数据目录下的`timescaledb/certs`。

### `timescaledb.passfile (string)`
指定存储密码的文件名，以及在连接到使用密码认证的数据节点时使用。

## 管理

### `timescaledb.restoring (bool)`
设置TimescaleDB为恢复模式。默认禁用。

### `timescaledb.license (string)`
根据使用的TimescaleDB许可证更改功能访问权限。例如，将`timescaledb.license`设置为`apache`将限制TimescaleDB仅使用Apache 2许可证下实现的功能。默认值为`timescale`，允许访问所有功能。

### `timescaledb.telemetry_level (enum)`
遥测设置级别。用于确定发送哪些遥测数据。可以设置为`off`或`basic`。默认为`basic`。

### `timescaledb.last_tuned (string)`
记录`timescaledb-tune`上次运行的时间。

### `timescaledb.last_tuned_version (string)`
记录运行时使用的`timescaledb-tune`版本。

[continuous-aggregates]: /use-timescale/:currentVersion:/continuous-aggregates/


