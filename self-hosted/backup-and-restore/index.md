---
标题: 备份与恢复
摘要: 学习如何备份及恢复你的 TimescaleDB 实例
产品: [自托管]
关键词: [备份，restore]
标签: [recovery]
---

import 考虑云服务 from "versionContent/_partials/_consider-cloud.mdx";

# 备份和恢复

TimescaleDB 利用 PostgreSQL 提供的可靠备份和恢复功能。您可以使用几种不同的机制来备份您的自托管 TimescaleDB 数据库：

*   使用 pg_dump 和 pg_restore 进行逻辑备份。
*   使用 `pg_basebackup` 或另一个工具进行[物理备份][物理备份]。
*   使用写前日志 (WAL) 归档的[持续物理备份][持续物理备份]（已弃用）。

[持续物理备份]: /self-hosted/:currentVersion:/backup-and-restore/docker-and-wale/
[物理备份]: /self-hosted/:currentVersion:/backup-and-restore/physical/
