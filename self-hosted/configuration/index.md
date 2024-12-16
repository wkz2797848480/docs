---
标题: 配置
摘要: 了解如何配置您的 TimescaleDB 实例
产品: [自托管]
关键词: [配置，设置]
---

以下是您提供的文章翻译成中文，并保留了超链接和格式，使用Markdown文档呈现：

# 配置

默认情况下，TimescaleDB使用默认的PostgreSQL服务器配置设置。然而，在某些情况下，这些设置并不合适，特别是当您拥有较大的服务器，使用更多的硬件资源，如CPU、内存和存储时。

*   [了解配置][config]以在开始使用之前理解其工作原理。
*   使用[TimescaleDB调优工具][tstune-conf]。
*   手动编辑`postgresql.conf`[配置文件][postgresql-conf]。
*   如果您在Docker容器中运行TimescaleDB，请在[Docker中配置][docker-conf]。
*   了解更多关于我们收集的[数据][telemetry]。

[config]: /self-hosted/:currentVersion:/configuration/about-configuration
[docker-conf]: /self-hosted/:currentVersion:/configuration/docker-config
[postgresql-conf]: /self-hosted/:currentVersion:/configuration/postgres-config
[telemetry]: /self-hosted/:currentVersion:/configuration/telemetry
[tstune-conf]: /self-hosted/:currentVersion:/configuration/timescaledb-tune
