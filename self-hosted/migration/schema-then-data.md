---
标题: 分别迁移模式和数据
摘要: 将你的 Timescale 数据和模式迁移至自托管的 TimescaleDB
产品: [自托管]
关键词: [数据迁移]
标签: [摄取]
---

import UsingParallelCopy from "versionContent/_partials/_migrate_using_parallel_copy.mdx";
import UsingPostgresCopy from "versionContent/_partials/_migrate_using_postgres_copy.mdx";
import PostSchemaEtal from "versionContent/_partials/_migrate_post_schema_caggs_etal.mdx";

# 分别迁移模式和数据

通过先迁移模式，然后迁移数据的方式来迁移较大的数据库。这种方法分别复制每个表或数据块，允许您在某个复制操作失败时中途重启。

<Highlight type="note">

对于较小的数据库，一次性迁移整个数据库可能更方便。更多信息，请参见[选择迁移方法][migration]部分。

</Highlight>

<Highlight type="warning">

这种方法不保留使用已删除数据计算的连续聚合。例如，如果您在一个月后删除原始数据，但保留连续聚合中的聚合数据一年，则连续聚合在迁移后将丢失任何比一个月更旧的数据。如果您必须保留使用已删除数据计算的连续聚合，请一次性迁移整个数据库。更多信息，请参见[选择迁移方法][migration]部分。

</Highlight>

迁移数据库的程序需要执行以下步骤：

*   [迁移模式预数据](#migrate-schema-pre-data)
*   [在Timescale中恢复超表](#restore-hypertables-in-timescale)
*   [从源数据库复制数据](#copy-data-from-the-source-database)
*   [将数据恢复到Timescale](#restore-data-into-timescale)
*   [迁移模式后数据](#migrate-schema-post-data)
*   [重新创建连续聚合](#recreate-continuous-aggregates)（可选）
*   [重新创建策略](#recreate-policies)（可选）
*   [更新表统计信息](#update-table-statistics)

<Highlight type="warning">

根据数据库大小和网络速度，涉及复制数据的步骤可能需要很长时间。您可以在这段时间内继续从源数据库读取，尽管性能可能会变慢。为了避免这个问题，从数据库分叉并从分叉中迁移您的数据。如果您在迁移期间写入源数据库中的表，新写入可能不会转移到Timescale。为了避免这个问题，请参见[迁移活动数据库][migration]部分。

</Highlight>

## 前提条件

开始之前，请检查您是否已经：

*   安装了PostgreSQL [`pg_dump`][pg_dump] 和 [`pg_restore`][pg_restore] 工具。
*   安装了用于连接PostgreSQL的客户端。这些说明使用 [`psql`][psql]，但任何客户端都可以。
*   在Timescale中创建了一个新的空数据库。更多信息，请参见[安装Timescale部分][install-selfhosted]。为您的数据库分配足够的空间以容纳所有数据。
*   检查您使用的任何其他PostgreSQL扩展是否与Timescale兼容。更多信息，请参见[兼容扩展列表][extensions]。安装您的其他PostgreSQL扩展。
*   检查您在Timescale和源数据库上运行的PostgreSQL是否为同一主版本。有关在源数据库上升级PostgreSQL的信息，请参阅[自托管TimescaleDB的升级说明][upgrading-postgresql-self-hosted]和管理服务TimescaleDB[upgrading-postgresql]。
*   检查您在目标和源数据库上运行的Timescale是否为同一主版本。更多信息，请参见[升级Timescale部分][upgrading-timescaledb]。

## 迁移模式预数据

从源数据库迁移您的预数据到自托管的TimescaleDB。这包括表和模式定义，以及有关序列、所有者和设置的信息。这不包括Timescale特定的模式。

<Procedure>

### 迁移模式预数据

1.  使用源数据库连接详情，将模式预数据从源数据库转储到 `dump_pre_data.bak` 文件中。排除Timescale特定的模式。如果系统提示输入密码，请使用您的源数据库凭据：

    ```bash
    pg_dump -U <SOURCE_DB_USERNAME> -W \
    -h <SOURCE_DB_HOST> -p <SOURCE_DB_PORT> -Fc -v \
    --section=pre-data --exclude-schema="_timescaledb*" \
    -f dump_pre_data.bak <DATABASE_NAME>
    ```

1.  使用您的Timescale连接详情，将转储的数据从 `dump_pre_data.bak` 文件恢复到您的Timescale数据库中。为了避免权限错误，请包含 `--no-owner` 标志：

    ```bash
    pg_restore -U tsdbadmin -W \
    -h <HOST> -p <PORT> --no-owner -Fc \
    -v -d tsdb dump_pre_data.bak
    ```

</Procedure>

## 在Timescale中恢复超表

预数据迁移后，您源数据库中的超表变成了Timescale中的常规PostgreSQL表。在Timescale中重新创建您的超表以恢复它们。

<Procedure>

### 在Timescale中恢复超表

1.  连接到您的Timescale数据库：

    ```sql
    psql "postgres://tsdbadmin:<PASSWORD>@<HOST>:<PORT>/tsdb?sslmode=require"
    ```

1.  恢复超表：

    ```sql
    SELECT create_hypertable(
       '<TABLE_NAME>',
	   by_range('<COLUMN_NAME>', INTERVAL '<CHUNK_INTERVAL>')
    );
    ```

</Procedure>

<Highlight type="note">
`by_range` 维度构建器是TimescaleDB 2.13的新增功能。
</Highlight>

## 从源数据库复制数据

恢复超表后，返回到源数据库逐个复制您的数据。

<Procedure>

### 从源数据库复制数据

1.  连接到您的源数据库：

    ```bash
    psql "postgres://<SOURCE_DB_USERNAME>:<SOURCE_DB_PASSWORD>@<SOURCE_DB_HOST>:<SOURCE_DB_PORT>/<SOURCE_DB_NAME>?sslmode=require"
    ```

1.  将第一个表的数据转储到 `.csv` 文件中：

    ```sql
    \COPY (SELECT * FROM <TABLE_NAME>) TO <TABLE_NAME>.csv CSV
    ```

    对于您想要迁移的每个表和超表重复此操作。

</Procedure>

<Highlight type="note">
如果您的表非常大，可以分多个部分迁移每个表。
按时间范围分割每个表，并单独复制每个范围。例如：

```sql
\COPY (SELECT * FROM <TABLE_NAME> WHERE time > '2021-11-01' AND time < '2011-11-02') TO <TABLE_NAME_DATE_RANGE>.csv CSV
```

</Highlight>

## 将数据恢复到Timescale

当您将数据复制到 `.csv` 文件后，可以通过从 `.csv` 文件复制来将其恢复到Timescale。有两种方法：使用常规PostgreSQL [`COPY`][copy]，或使用TimescaleDB [`timescaledb-parallel-copy`][timescaledb-parallel-copy] 函数。在测试中，`timescaledb-parallel-copy` 速度提高了16%。`timescaledb-parallel-copy` 工具不是默认包含的。您必须安装该函数。

<Highlight type="important">
因为 `COPY` 会解压缩数据，所以源数据库中的任何压缩数据现在都以未压缩的形式存储在您的 `.csv` 文件中。如果您为压缩数据提供了Timescale存储空间，未压缩的数据可能会占用太多存储空间。为了避免这个问题，请在复制时定期重新压缩您的数据。有关压缩的更多信息，请参见[压缩部分](https://docs.timescale.com/use-timescale/latest/compression/)。 
</Highlight>

<UsingParallelCopy />

<UsingPostgresCopy />

<PostSchemaEtal />

[copy]: https://www.postgresql.org/docs/9.2/sql-copy.html 
[extensions]: /use-timescale/:currentVersion:/extensions/
[install-selfhosted]: /self-hosted/:currentVersion:/install/
[pg_dump]: https://www.postgresql.org/docs/current/app-pgdump.html 
[pg_restore]: https://www.postgresql.org/docs/current/app-pgrestore.html 
[psql]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql/
[timescaledb-parallel-copy]: https://github.com/timescale/timescaledb-parallel-copy 
[upgrading-postgresql]: https://kb-managed.timescale.com/en/articles/5368016-perform-a-postgresql-major-version-upgrade 
[upgrading-postgresql-self-hosted]: /self-hosted/:currentVersion:/upgrades/upgrade-pg/
[upgrading-timescaledb]: /self-hosted/:currentVersion:/upgrades/major-upgrade/
[migration]: /migrate/:currentVersion:/

