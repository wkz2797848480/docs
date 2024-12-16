---
标题：升级 PostgreSQL
摘要：将 PostgreSQL 升级到一个新版本。
产品：[自托管]
关键词：[升级，PostgreSQL，版本，兼容性]
---

import PlanUpgrade from "versionContent/_partials/_plan_upgrade.mdx";
import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";
import PlanMigrationPath from "versionContent/_partials/_migrate_self_postgres_plan_migration_path.mdx";

# 升级 PostgreSQL

TimescaleDB 是 PostgreSQL 的一个扩展。确保您升级到兼容版本的 TimescaleDB 和 PostgreSQL。

<ConsiderCloud />

## 先决条件

<PlanUpgrade />

## 规划您的升级路径

<PlanMigrationPath />

## 升级您的 PostgreSQL 实例

您使用 [`pg_upgrade`][pg_upgrade] 来原地升级 PostgreSQL。`pg_upgrade` 允许您保留当前 PostgreSQL 安装的数据文件，同时将新的 PostgreSQL 二进制运行时绑定到它们。

<Procedure>

1. **查找 PostgreSQL 二进制文件的位置**

   设置 `OLD_BIN_DIR` 环境变量为包含 `postgres` 二进制文件的文件夹。
   例如，`which postgres` 返回类似于 `/usr/lib/postgresql/16/bin/postgres`。
   ```bash
   export OLD_BIN_DIR=/usr/lib/postgresql/16/bin
   ``` 

1. **设置您的连接字符串**

   此变量保存要升级的数据库的连接信息：

   ```bash
   export SOURCE="postgres://<user>:<password>@<source host>:<source port>/<db_name>"
   ```

1. **检索 PostgreSQL 数据文件夹的位置**

    设置 `OLD_DATA_DIR` 环境变量为以下命令返回的值：
    ```shell
    psql -d "$SOURCE" -c "SHOW data_directory ;" 
    ```
    PostgreSQL 返回类似于：
    ```shell
    ----------------------------
    /home/postgres/pgdata/data
    (1 row)
    ```        

1. **选择 PostgreSQL 二进制和数据文件夹的新位置**

   例如：
   ```shell
   export NEW_BIN_DIR=/usr/lib/postgresql/17/bin
   export NEW_DATA_DIR=/home/postgres/pgdata/data-17
   ```        
1. 使用 psql，执行升级：

    ```sql
    pg_upgrade -b $OLD_BIN_DIR -B $NEW_BIN_DIR -d $OLD_DATA_DIR -D $NEW_DATA_DIR
    ```

</Procedure>

如果您正在将数据迁移到 PostgreSQL 的新物理实例，您可以使用 `pg_dump` 和 `pg_restore` 来转储旧数据库中的数据，然后将其恢复到新的、已升级的数据库中。更多信息，请参阅 [备份和恢复部分][backup]。

[backup]: /self-hosted/:currentVersion:/backup-and-restore/
[pg-relnotes]: https://www.postgresql.org/docs/release/ 
[pg_upgrade]: https://www.postgresql.org/docs/current/static/pgupgrade.html 
[postgres-breaking-change]: https://www.postgresql.org/about/news/postgresql-172-166-1510-1415-1318-and-1222-released-2965/ 
[upgrade-pg]: /self-hosted/:currentVersion:/upgrades/upgrade-pg/#upgrade-postgresql
