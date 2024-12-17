---
标题: 能源时间序列数据教程 —— 设置数据集
摘要: 设置数据集，以便能够查询时间序列数据。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [教程，创建，数据集]
标签: [教程，初学者]
布局组件: [大尺寸的上一页 / 下一页按钮]
内容分组: 分析能源消耗数据
---

import CreateAndConnect from "versionContent/_partials/_cloud-create-connect-tutorials.mdx";
import CreateHypertableEnergy from "versionContent/_partials/_create-hypertable-energy.mdx";
import AddDataEnergy from "versionContent/_partials/_add-data-energy.mdx";
import GrafanaConnect from "versionContent/_partials/_grafana-connect.mdx";
import CreateCaggs from "versionContent/_partials/_caggs-intro.mdx";

# 设置数据库

本教程使用名为`metrics`的超表中一年多的能耗数据。

<Collapsible heading="创建Timescale服务并连接到您的服务" defaultExpanded={false}>

<CreateAndConnect/>

</Collapsible>

<Collapsible heading="数据集" defaultExpanded={false}>

本教程使用了典型家庭一年多的能耗数据。您可以使用这些数据来分析能耗模式。

<CreateHypertableEnergy />

<AddDataEnergy />

</Collapsible>

<Collapsible heading="数据降采样" defaultExpanded={false}>

<CreateCaggs />

## 创建连续聚合

<Procedure>

### 为每日和每小时的能耗创建连续聚合

1.  创建一个连续聚合`kwh_day_by_day`，用于按日计算能耗：

    ```sql
    CREATE MATERIALIZED VIEW kwh_day_by_day(time, value)
        with (timescaledb.continuous) as
    SELECT time_bucket('1 day', created, 'Europe/Berlin') AS "time",
            round((last(value, created) - first(value, created)) * 100.) / 100. AS value
    FROM metrics
    WHERE type_id = 5
    GROUP BY 1;
    ```

2.  添加刷新策略以保持连续聚合最新：

     ```sql
     SELECT add_continuous_aggregate_policy('kwh_day_by_day',
        start_offset => NULL,
        end_offset => INTERVAL '1 hour',
        schedule_interval => INTERVAL '1 hour');
     ```

3.  创建一个连续聚合`kwh_hour_by_hour`，用于按小时计算能耗：

    ```sql
    CREATE MATERIALIZED VIEW kwh_hour_by_hour(time, value)
      with (timescaledb.continuous) as
    SELECT time_bucket('01:00:00', metrics.created, 'Europe/Berlin') AS "time",
           round((last(value, created) - first(value, created)) * 100.) / 100. AS value
    FROM metrics
    WHERE type_id = 5
    GROUP BY 1;
    ```

4.  添加刷新策略以保持连续聚合最新：

     ```sql
     SELECT add_continuous_aggregate_policy('kwh_hour_by_hour',
        start_offset => NULL,
        end_offset => INTERVAL '1 hour',
        schedule_interval => INTERVAL '1 hour');
     ```

5.  您可以确认连续聚合已创建：

    ```sql
    SELECT view_name, format('%I.%I', materialization_hypertable_schema,materialization_hypertable_name) AS materialization_hypertable
    FROM timescaledb_information.continuous_aggregates;
    ```

    您应该看到以下内容：

    ```sql
     view_name     |            materialization_hypertable
    ------------------+--------------------------------------------------
     kwh_day_by_day   | _timescaledb_internal._materialized_hypertable_2
     kwh_hour_by_hour | _timescaledb_internal._materialized_hypertable_3

    ```

</Procedure>

</Collapsible>

<Collapsible heading="连接到Grafana" defaultExpanded={false}>

本教程中的查询适合在Grafana中进行可视化。如果您想可视化查询结果，请将您的Grafana账户连接到能耗数据集。

<GrafanaConnect />

</Collapsible>


</Collapsible>
