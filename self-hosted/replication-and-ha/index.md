---
标题: 高可用性
摘要: 了解高可用性相关知识。
产品: [自托管]
关键词: [高可用性]
---

import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";

# 复制和高可用性

PostgreSQL依赖复制来实现高可用性、故障转移以及在多个节点间平衡读取负载。复制确保写入主PostgreSQL数据库的数据在一到多个节点上被镜像。由于有多个节点拥有主数据库的精确副本，因此在主服务器发生故障或中断时，主数据库可以被替换为副本节点。副本节点也可以作为只读数据库使用，也称为读取副本，允许通过在多个节点间分散读取查询量来水平扩展读取。

*   [了解高可用性][about-ha]，在开始使用之前了解其工作原理。
*   [配置复制][replication-enable]。

<考虑云环境 />

[about-ha]: /self-hosted/:currentVersion:/replication-and-ha/about-ha/
[replication-enable]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication/
