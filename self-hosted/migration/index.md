---
标题: 将你的 PostgreSQL 数据库迁移至自托管的 TimescaleDB
摘要: 将现有的 PostgreSQL 数据库迁移至 Timescale
产品: [自托管]
关键词: [数据迁移，自托管，PostgreSQL，关系型数据库服务（RDS）]
标签: [摄取，迁移，关系型数据库服务（RDS）]
---

# 将您的PostgreSQL数据库迁移到自托管的TimescaleDB

您可以将现有的PostgreSQL数据库迁移到自托管的Timescale安装。

迁移数据有几种方法：

*   如果您要迁移的数据库小于100&nbsp;GB，[一次性迁移整个数据库][migrate-entire]：
    这种方法直接传输所有数据和模式，包括Timescale特定功能。您的超表、连续聚合和策略在新的Timescale数据库中自动可用。
*   对于大于100&nbsp;GB的数据库，[分别迁移您的模式和数据][migrate-separately]：这种方法让您逐个迁移表，以便更容易地恢复失败。如果迁移中途失败，您可以从失败点重新开始，而不是从头开始。但是，Timescale特定功能不会自动迁移。请按照说明恢复您的超表、连续聚合和策略。
*   如果您需要将数据从PostgreSQL表移动到现有Timescale数据库中的超表，[在同一数据库内迁移][migrate-same-db]：这种方法假设您已经在与现有表相同的数据库实例中设置了Timescale。
*   如果您有InfluxDB数据库中的数据，[使用Outflux迁移][outflux]：
    Outflux直接将导出的数据管道传输到Timescale，并管理模式发现、验证和创建。Outflux与早期版本的InfluxDB兼容。它不适用于InfluxDB版本2及更高版本。

## 选择迁移方法

您选择哪种方法取决于数据库大小、网络上传和下载速度、现有的连续聚合以及对失败恢复的容忍度。

<Highlight type="note">
如果您是从Amazon RDS服务迁移，Amazon会收取数据转移出服务的费用。即使迁移失败，您也可能被Amazon收取所有数据流出的费用。
</Highlight>

如果您的数据库小于100&nbsp;GB，请选择一次性迁移整个数据库。您也可以使用这种方法迁移更大的数据库，但复制过程必须持续运行，可能需要几天或几周。如果复制被中断，则需要重新开始该过程。如果您认为复制可能会中断，请选择分别迁移模式和数据。

<Highlight type="warning">
分别迁移模式和数据不会保留使用已删除数据计算的连续聚合。例如，如果您在一个月后删除原始数据，但保留连续聚合中的聚合数据一年，那么在迁移后连续聚合将丢失任何比一个月更旧的数据。如果您必须保留使用已删除数据计算的连续聚合，请无论数据库大小如何，都选择一次性迁移整个数据库。
</Highlight>

如果您不确定使用哪种方法，尝试一次性复制整个数据库以估计所需时间。如果时间估计非常长，请停止迁移并切换到另一种方法。

## 迁移活动数据库

如果您的数据库正在积极地摄取数据，请采取预防措施，确保您的Timescale数据库包含迁移过程中摄取的数据。首先在源数据库和目标数据库上并行运行摄取。这确保最新数据被写入两个数据库。然后使用两种迁移方法之一回填您的数据。

[migrate-entire]: /self-hosted/:currentVersion:/migration/entire-database/
[migrate-separately]: /self-hosted/:currentVersion:/migration/schema-then-data/
[migrate-same-db]: /self-hosted/:currentVersion:/migration/same-db/
[outflux]: /self-hosted/:currentVersion:/migration/migrate-influxdb/
