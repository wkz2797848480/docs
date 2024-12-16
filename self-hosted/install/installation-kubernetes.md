---
标题: 在 Kubernetes 上安装 TimescaleDB
摘要: 在 Kubernetes 上安装自托管的 TimescaleDB
产品: [自托管]
关键词: [安装，自托管，Kubernetes]
---

import Skip from "versionContent/_partials/_selfhosted_cta.mdx";

# 在Kubernetes上安装TimescaleDB

TimescaleDB可以使用TimescaleDB Docker容器映像在Kubernetes内部运行。
过去，Timescale维护Helm图表来管理Kubernetes部署，但现在我们推荐Kubernetes用户依赖于以下出色的PostgreSQL Kubernetes操作符来简化安装、配置和生命周期管理。

<跳过 />

我们的社区成员告诉我们以下操作符工作良好：

- [StackGres][stackgres]（包括TimescaleDB映像）
- [Postgres Operator (Patroni)][patroni] 
- [PGO][pgo]
- [CloudNativePG][cnpg]

[stackgres]: https://github.com/ongres/stackgres 
[patroni]: https://github.com/zalando/postgres-operator 
[pgo]: https://github.com/CrunchyData/postgres-operator 
[cnpg]: https://github.com/cloudnative-pg/cloudnative-pg



