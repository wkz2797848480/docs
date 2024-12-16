---
标题: 从同一个 PostgreSQL 实例将数据迁移至 Timescale
摘要: 将数据从常规的 PostgreSQL 表迁移至 Timescale 超表
产品: [自托管]
关键词: [数据迁移，PostgreSQL]
标签: [导入]
---

# 从同一PostgreSQL实例迁移数据到Timescale

您可以将常规PostgreSQL表中的数据迁移到Timescale的超表中。这种方法假设您已经在与现有表相同的数据库实例中设置了Timescale。

## 前提条件

开始之前，请确保您已经[安装并设置][install]了Timescale。

您还需要一个包含现有数据的表。在这个例子中，源表被命名为`old_table`。请将表名替换为您实际的表名。例子中还将目标表命名为`new_table`，但您可能想要使用一个更具描述性的名称。

## 迁移数据

在同一数据库内将数据迁移到Timescale。

<Procedure>

## 迁移数据

1.  基于现有表创建一个新表。您可以同时创建索引，这样您就不需要手动重新创建它们。或者，您可以在不包含索引的情况下创建表，这会使数据迁移更快。

    <Terminal>

    <tab label="包含索引">

    ```bash
    CREATE TABLE new_table (
        LIKE old_table INCLUDING DEFAULTS INCLUDING CONSTRAINTS INCLUDING INDEXES
    );
    ```

    </tab>

    <tab label="不包含索引">

    ```bash
    CREATE TABLE new_table (
        LIKE old_table INCLUDING DEFAULTS INCLUDING CONSTRAINTS EXCLUDING INDEXES
    );
    ```

    </tab>

    </Terminal>

1.  使用[`create_hypertable`][create_hypertable]函数将新表转换为超表。将`ts`替换为您表中包含时间值的列名。

    ```sql
    SELECT create_hypertable('new_table', 'ts');
    ```

    <Highlight type="note">
    `by_range`维度构建器是TimescaleDB 2.13的新增功能。
    </Highlight>

1.  从旧表向新表插入数据。

    ```sql
    INSERT INTO new_table
      SELECT * FROM old_table;
    ```

1.  如果您在创建新表时没有包含索引，请现在重新创建您的索引。

</Procedure>

[create_hypertable]: /api/:currentVersion:/hypertable/create_hypertable/
[install]: /getting-started/latest/

