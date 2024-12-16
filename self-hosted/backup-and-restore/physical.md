---
标题: 物理备份
摘要: 如何对您的 TimescaleDB 实例进行物理备份
产品: [自托管]
关键字: [备份]
标签: [restore，recovery，物理备份]
---

import 考虑云服务 from "versionContent/_partials/_consider-cloud.mdx";

# 物理备份

对于完整的实例物理备份（特别是对于启动新的[复制][replication-tutorial]非常有用），[`pg_basebackup`][postgres-pg_basebackup]适用于所有TimescaleDB安装类型。您还可以使用任何多个外部备份和恢复管理器，例如[`pg_backrest`][pg-backrest]或[`barman`][pg-barman]。对于持续的物理备份，您可以使用[`wal-e`][wale]，尽管这种方法现在已被弃用。这些工具都允许您对整个实例进行在线物理备份，许多工具还提供增量备份和其他自动化选项。

[pg-backrest]: https://pgbackrest.org/ 
[pg-barman]: https://www.pgbarman.org/ 
[postgres-pg_basebackup]: https://www.postgresql.org/docs/current/app-pgbasebackup.html 
[replication-tutorial]: /self-hosted/:currentVersion:/replication-and-ha/
[wale]: /self-hosted/:currentVersion:/backup-and-restore/docker-and-wale/

