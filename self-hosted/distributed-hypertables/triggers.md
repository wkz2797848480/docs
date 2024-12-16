---
标题: 在分布式超表上使用触发器
摘要: 如何在分布式超表上设置触发器
产品: [自托管]
关键词: [分布式超表，触发器，多节点]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 在分布式超表上使用触发器

分布式超表上的触发器与标准超表上的触发器工作方式大致相同，并具有相同的限制。但由于数据分布在多个节点上，存在一些差异：

*   行级触发器在插入行的数据节点上触发。触发器必须在数据存储的位置触发，因为`BEFORE`和`AFTER`行触发器需要访问存储的数据。接入节点上的块不包含任何数据，因此它们没有触发器。
*   语句级触发器在每个受影响的节点上触发一次，包括接入节点。例如，如果一个分布式超表包含3个数据节点，插入2行数据将执行语句级触发器在接入节点和1个或2个数据节点上，这取决于行是否进入相同的节点或不同的节点。
*   复制因子大于1进一步导致触发器在多个节点上触发。每个副本节点都会触发触发器。

## 在分布式超表上创建触发器

通过使用[`CREATE TRIGGER`][create-trigger]像平常一样在分布式超表上创建触发器。触发器及其执行的函数会自动在每个数据节点上创建。如果触发器函数引用了任何其他函数或对象，需要在创建触发器之前在所有节点上存在它们。

<Procedure>

### 在分布式超表上创建触发器

1.  如果您的触发器需要引用另一个函数或对象，请使用[`distributed_exec`][distributed_exec]在所有节点上创建该函数或对象。
2.  在接入节点上创建触发器函数。此示例创建一个虚拟触发器，触发时会发出通知'trigger fired'：

    ```sql
    CREATE OR REPLACE FUNCTION my_trigger_func()
    RETURNS TRIGGER LANGUAGE PLPGSQL AS
    $BODY$
    BEGIN
    RAISE NOTICE 'trigger fired';
    RETURN NEW;
    END
    $BODY$;
    ```

3.  在接入节点上创建触发器本身。此示例导致在向超表`hyper`插入行时触发触发器。注意您不需要手动在数据节点上创建触发器。这是自动为您完成的。

    ```sql
    CREATE TRIGGER my_trigger
    AFTER INSERT ON hyper
    FOR EACH ROW
    EXECUTE FUNCTION my_trigger_func();
    ```

</Procedure>

## 避免多次处理触发器

如果您有语句级触发器，或者复制因子大于1，触发器会多次触发。为了避免重复触发，您可以设置触发器函数来检查它正在哪个数据节点上执行。

例如，编写一个触发器函数，在接入节点上与数据节点上发出不同的通知：

```sql
CREATE OR REPLACE FUNCTION my_trigger_func()
    RETURNS TRIGGER LANGUAGE PLPGSQL AS
$BODY$
DECLARE
    is_access_node boolean;
BEGIN
    SELECT is_distributed INTO is_access_node
    FROM timescaledb_information.hypertables
    WHERE hypertable_name = <TABLE_NAME>
    AND hypertable_schema = <TABLE_SCHEMA>;

    IF is_access_node THEN
       RAISE NOTICE 'trigger fired on the access node';
    ELSE
       RAISE NOTICE 'trigger fired on a data node';
    END IF;

    RETURN NEW;
END
$BODY$;
```

