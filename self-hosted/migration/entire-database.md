---
标题: 一次性迁移整个数据库
摘要: 一次性将整个 Timescale 数据库迁移至自托管的 TimescaleDB
产品: [自托管]
关键词: [数据迁移]
标签: [摄取]
---

# 一次性迁移整个数据库

对于小于100 GB的小型数据库，可以通过一次性转储和恢复整个数据库来进行迁移。
对于更大的数据库，请考虑[分别迁移您的模式和数据][migrate-separately]。

<Highlight type="warning">

根据数据库大小和网络速度，迁移可能需要很长时间。
在此过程中，您可以继续从源数据库中读取数据，但性能可能会变慢。
为了避免这个问题，可以从数据库中分叉出来进行迁移。
如果您在迁移过程中写入源数据库中的表，新写入的数据可能不会传输到 Timescale。
为了避免这个问题，请参见[实时迁移][live-migration]。

</Highlight>

## 前提条件

开始之前，请检查您是否已经：

*   安装了 PostgreSQL [`pg_dump`][pg_dump] 和 [`pg_restore`][pg_restore] 工具。
*   安装了用于连接 PostgreSQL 的客户端。这些说明使用 [`psql`][psql]，但任何客户端都可以。
*   在 Timescale 中创建了一个新的空数据库。更多信息，请参见[安装 Timescale部分][install-selfhosted-timescale]。
   为您的数据预配足够的空间。
*   检查您使用的任何其他 PostgreSQL 扩展是否与 Timescale 兼容。
   更多信息，请参见[兼容扩展列表][extensions]。
   安装您的其他 PostgreSQL 扩展。
*   检查您的目标和源数据库是否运行相同主版本的 PostgreSQL。
   有关在源数据库上升级 PostgreSQL 的信息，请参阅[自托管 TimescaleDB 的升级说明][upgrading-postgresql-self-hosted]。
*   检查您的目标和源数据库是否运行相同主版本的 Timescale。
   更多信息，请参阅[升级 Timescale部分][upgrading-timescaledb]。

<Highlight type="note">

为了加快迁移速度，可以压缩您的数据。
您可以压缩任何不在当前插入、更新或删除的数据块。
迁移完成后，可以根据正常操作的需要对数据块进行解压缩。
有关压缩和解压缩的更多信息，请参阅[压缩][compression]。

</Highlight>

<Procedure>

### 一次性迁移整个数据库

1.  使用源数据库连接详情，将所有数据从源数据库转储到 `dump.bak` 文件中。
   如果系统提示输入密码，请使用您的源数据库凭据：

   ```bash
   pg_dump -U <SOURCE_DB_USERNAME> -W \
   -h <SOURCE_DB_HOST> -p <SOURCE_DB_PORT> -Fc -v \
   -f dump.bak <SOURCE_DB_NAME>
   ```

1.  使用您的 Timescale 连接详情连接到您的 Timescale 数据库。
   当系统提示输入密码时，请使用您的 Timescale 凭据：

   ```bash
   psql “postgres://tsdbadmin:<PASSWORD>@<HOST>:<PORT>/tsdb?sslmode=require”
   ```

1.  通过使用 [`timescaledb_pre_restore`][timescaledb_pre_restore] 停止后台工作进程，为您的 Timescale 数据库准备数据恢复：

   ```sql
   SELECT timescaledb_pre_restore();
   ```

1.  在命令提示符下，将 `dump.bak` 文件中转储的数据恢复到您的 Timescale 数据库中，使用您的 Timescale 连接详情。
   为了避免权限错误，请包含 `--no-owner` 标志：

   ```bash
   pg_restore -U tsdbadmin -W \
   -h <CLOUD_HOST> -p <CLOUD_PORT> --no-owner \
   -Fc -v -d tsdb dump.bak
   ```

1.  在 `psql` 提示符下，通过使用 [`timescaledb_post_restore`][timescaledb_post_restore] 命令将您的 Timescale 数据库恢复到正常操作：

   ```sql
   SELECT timescaledb_post_restore();
   ```

1.  通过在整个数据集上运行 [`ANALYZE`][analyze] 来更新您的表统计信息：

   ```sql
   ANALYZE;
   ```

</Procedure>

[analyze]: https://www.postgresql.org/docs/10/sql-analyze.html 
[extensions]: /use-timescale/:currentVersion:/extensions/
[install-selfhosted-timescale]: /self-hosted/:currentVersion:/install/
[migrate-separately]: /self-hosted/:currentVersion:/migration/schema-then-data/
[pg_dump]: https://www.postgresql.org/docs/current/app-pgdump.html 
[pg_restore]: https://www.postgresql.org/docs/current/app-pgrestore.html 
[psql]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql/
[timescaledb_pre_restore]: /api/:currentVersion:/administration/#timescaledb_pre_restore
[timescaledb_post_restore]: /api/:currentVersion:/administration/#timescaledb_post_restore
[upgrading-postgresql-self-hosted]: /self-hosted/:currentVersion:/upgrades/upgrade-pg/
[upgrading-timescaledb]: /self-hosted/:currentVersion:/upgrades/major-upgrade/
[live-migration]: /migrate/:currentVersion:/live-migration/
[compression]: /use-timescale/:currentVersion:/compression/
