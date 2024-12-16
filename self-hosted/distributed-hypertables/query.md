---
标题: 查询分布式超表中的数据
摘要: 如何查询分布式超表中的数据
产品: [自托管]
关键词: [分布式超表，多节点，查询]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 在分布式超表中查询数据

您可以像查询标准超表或PostgreSQL表一样查询分布式超表。更多信息，请参阅[写入数据][write]部分。

当接入节点能够将事务下推到数据节点时，查询性能最佳。为确保接入节点能够下推事务，请检查[`enable_partitionwise_aggregate`][enable_partitionwise_aggregate]设置是否为接入节点设置为`on`。默认情况下，它处于`off`状态。

如果您希望在分布式超表上使用连续聚合，请参阅[连续聚合][caggs]部分以获取更多信息。

[caggs]: /use-timescale/:currentVersion:/continuous-aggregates/
[enable_partitionwise_aggregate]: https://www.postgresql.org/docs/current/runtime-config-query.html 
[write]: /use-timescale/:currentVersion:/write-data/
