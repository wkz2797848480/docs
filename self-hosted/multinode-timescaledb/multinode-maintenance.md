---
标题: 多节点维护任务
摘要: 如何维护你的多节点实例
产品: [自托管]
关键词: [多节点，维护]
标签: [管理]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 多节点维护任务

为了有效维护分布式多节点设置，需要进行各种维护活动。如果需要，您可以使用`cron`或其他调度系统在数据库外部定期运行以下维护工作。同时确保为包含分布式超表的每个数据库分别安排作业。

## 维护分布式事务

分布式事务跨多个数据节点运行，如果数据节点重新启动或遇到临时问题，可能会保持未完成状态。访问节点保留分布式事务的日志，以便尚未完成其部分分布式事务的节点在可用时可以稍后完成。这个事务日志需要定期清理以移除已完成的事务，并完成尚未完成的事务。我们强烈建议您配置访问节点定期运行维护作业，清理任何未完成的分布式事务。

自定义维护作业可以作为用户定义的操作运行。例如：

<Tabs title="自定义维护作业">
<Tab title="TimescaleDB >= 2.12">

```sql
CREATE OR REPLACE PROCEDURE data_node_maintenance(job_id int, config jsonb)
LANGUAGE SQL AS
$$
    SELECT _timescaledb_functions.remote_txn_heal_data_node(fs.oid)
    FROM pg_foreign_server fs, pg_foreign_data_wrapper fdw
    WHERE fs.srvfdw = fdw.oid
    AND fdw.fdwname = 'timescaledb_fdw';
$$;

SELECT add_job('data_node_maintenance', '5m');
```

</Tab>

<Tab title="TimescaleDB < 2.12">

```sql
CREATE OR REPLACE PROCEDURE data_node_maintenance(job_id int, config jsonb)
LANGUAGE SQL AS
$$
    SELECT _timescaledb_internal.remote_txn_heal_data_node(fs.oid)
    FROM pg_foreign_server fs, pg_foreign_data_wrapper fdw
    WHERE fs.srvfdw = fdw.oid
    AND fdw.fdwname = 'timescaledb_fdw';
$$;

SELECT add_job('data_node_maintenance', '5m');
```

</Tab>
</Tabs>

## 分布式超表的统计信息

在分布式超表上，需要保持表统计信息的更新。这允许您有效地计划查询。由于分布式超表的特性，您不能使用`auto-vacuum`工具收集统计信息。相反，您可以定期使用维护作业显式分析分布式超表，如下所示：

```sql
CREATE OR REPLACE PROCEDURE distributed_hypertables_analyze(job_id int, config jsonb)
LANGUAGE plpgsql AS
$$
DECLARE r record;
BEGIN
FOR r IN SELECT hypertable_schema, hypertable_name
              FROM timescaledb_information.hypertables
              WHERE is_distributed ORDER BY 1, 2
LOOP
EXECUTE format('ANALYZE %I.%I', r.hypertable_schema, r.hypertable_name);
END LOOP;
END
$$;

SELECT add_job('distributed_hypertables_analyze', '12h');
```

如果您喜欢，可以将此示例中的作业合并为一个单一的维护作业。然而，分析分布式超表应该比远程事务修复活动做得不那么频繁。这是因为前者每次可能会分析大量的远程数据块，如果调用得太频繁，可能会很昂贵。

