---
标题: 降级到 TimescaleDB 的旧版本
摘要: 将自托管的 TimescaleDB 降级到之前的次版本
产品: [自托管]
关键词: [升级]
---

import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";

# 降级TimescaleDB到次要版本

如果您升级到新的TimescaleDB版本并遇到问题，您可以回滚到之前安装的版本。这与次要升级的工作方式相同。

降级并不支持所有版本。通常，支持补丁版本之间以及连续次要版本之间的降级。例如，您可以从TimescaleDB 2.5.2降级到2.5.1，或者从2.5.0降级到2.4.2。要检查您是否可以从特定版本降级，请查看[发布说明][relnotes]。

<ConsiderCloud />

## 计划您的降级

您可以就地降级您的现场TimescaleDB安装。这意味着您不需要转储和恢复您的数据。然而，提前计划降级仍然很重要。

在您降级之前：

*   阅读您要降级的TimescaleDB版本的[发布说明][relnotes]。
*   检查您当前运行的PostgreSQL版本。在开始TimescaleDB降级之前，您可能需要[升级到最新的PostgreSQL版本][upgrade-pg]。
*   [执行数据库备份][backup]。尽管TimescaleDB降级是就地进行的，但降级是一个侵入性操作。始终确保您手头有备份，并在灾难发生时能够读取备份。

## 降级TimescaleDB到以前的次要版本

此降级使用PostgreSQL `ALTER EXTENSION`函数降级到TimescaleDB扩展的以前版本。TimescaleDB支持在同一PostgreSQL实例的不同数据库上拥有不同扩展版本。这允许您在不同数据库上独立升级和降级扩展。在每个数据库上运行`ALTER EXTENSION`函数以单独降级它们。

<Highlight type="important">

降级脚本针对单步降级进行了测试和支持。也就是说，从当前版本降级到上一个次要版本。如果您在升级和降级之间对数据库进行了更改，降级可能无法工作。

</Highlight>

<Procedure>

1. **设置您的连接字符串**

   此变量保存要升级的数据库的连接信息：

   ```bash
   export SOURCE="postgres://<user>:<password>@<source host>:<source port>/<db_name>"
   ```

2. **连接到您的数据库实例**
    ```shell
    psql -X -d $SOURCE
    ```

   `-X`标志防止任何`.psqlrc`命令意外触发会话启动时加载先前的TimescaleDB版本。

1. **降级TimescaleDB扩展** 
    这必须是您在当前会话中执行的第一个命令：

    ```sql
    ALTER EXTENSION timescaledb UPDATE TO '<PREVIOUS_VERSION>';
    ```

    例如：

    ```sql
    ALTER EXTENSION timescaledb UPDATE TO '2.17.0';
    ```

1. **检查您是否已降级到正确的TimescaleDB版本**

    ```sql
    \dx timescaledb;
    ```
   Postgres返回类似于：
    ```shell
    Name     | Version | Schema |                                      Description                                      
    -------------+---------+--------+---------------------------------------------------------------------------------------
    timescaledb | 2.17.0  | public | Enables scalable inserts and complex queries for time-series data (Community Edition)
    ```

</Procedure>

[backup]: /self-hosted/:currentVersion:/backup-and-restore/
[relnotes]: https://github.com/timescale/timescaledb/releases 
[upgrade-pg]: /self-hosted/:currentVersion:/upgrades/upgrade-pg/
