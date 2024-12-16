---
标题: 在分布式超表中创建外键
摘要: 向分布式超表的节点添加外键
产品: [自托管]
关键词: [分布式超表，外键]
标签: [约束条件]
---

import 多节点已弃用 from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 在分布式超表中创建外键

分布式超表所引用的表和值必须存在于接入节点和所有数据节点上。要创建一个从分布式超表到引用表的外键，请使用[`distributed_exec`][distributed_exec]首先在所有节点上创建引用表。

<Procedure>

## 在分布式超表中创建外键

1.  在接入节点上创建引用表。
2.  使用[`distributed_exec`][distributed_exec]在所有数据节点上创建相同的表，并用正确的数据更新它。
3.  从您的分布式超表到您的引用表创建一个外键。

</Procedure>

[distributed_exec]: /api/:currentVersion:/distributed-hypertables/distributed_exec/

