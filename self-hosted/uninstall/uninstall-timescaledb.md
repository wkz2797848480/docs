---
标题: 卸载 TimescaleDB
摘要: 卸载 TimescaleDB
产品: [自托管]
关键词: [卸载]
---

# 卸载TimescaleDB

PostgreSQL被设计为易于扩展。加载到数据库中的扩展可以像内置功能一样工作。TimescaleDB扩展了PostgreSQL以处理时间序列数据，为PostgreSQL提供了现代数据密集型应用所需的高性能、可扩展性和分析能力。如果您通过Homebrew或MacPorts安装了TimescaleDB，您可以在不卸载PostgreSQL的情况下卸载它。

<Procedure>

## 使用Homebrew卸载TimescaleDB

1.  在`psql`提示符下，移除TimescaleDB扩展：

    ```sql
    DROP EXTENSION timescaledb;
    ```

1.  在命令提示符下，从`postgresql.conf`配置文件中的`shared_preload_libraries`移除`timescaledb`：

    ```bash
    nano /opt/homebrew/var/postgresql@14/postgresql.conf
    shared_preload_libraries = ''
    ```

1.  保存`postgresql.conf`文件的更改。

1.  重启PostgreSQL：

    ```bash
    brew services restart postgresql
    ```

1.  通过在`psql`提示符下使用`\dx`命令检查TimescaleDB扩展是否已卸载。输出类似于：

    ```sql
    tsdb-# \dx
                                          List of installed extensions
        Name     | Version |   Schema   |                            Description
    -------------+---------+------------+-------------------------------------------------------------------
     plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language
    (1 row)
    ```

1.  卸载TimescaleDB：

    ```bash
    brew uninstall timescaledb
    ```

1.  移除所有依赖项和相关文件：

    ```bash
    brew remove timescaledb
    ```

</Procedure>

<Procedure>

## 使用MacPorts卸载TimescaleDB

1.  在`psql`提示符下，移除TimescaleDB扩展：

    ```sql
    DROP EXTENSION timescaledb;
    ```

1.  在命令提示符下，从`postgresql.conf`配置文件中的`shared_preload_libraries`移除`timescaledb`：

    ```bash
    nano /opt/homebrew/var/postgresql@14/postgresql.conf
    shared_preload_libraries = ''
    ```

1.  保存`postgresql.conf`文件的更改。

1.  重启PostgreSQL：

    ```bash
    port reload postgresql
    ```

1.  通过在`psql`提示符下使用`\dx`命令检查TimescaleDB扩展是否已卸载。输出类似于：

    ```sql
    tsdb-# \dx
                                          List of installed extensions
        Name     | Version |   Schema   |                            Description
    -------------+---------+------------+-------------------------------------------------------------------
     plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language
    (1 row)
    ```

1.  卸载TimescaleDB和相关依赖项：

    ```bash
    port uninstall timescaledb --follow-dependencies
    ```

</Procedure>

