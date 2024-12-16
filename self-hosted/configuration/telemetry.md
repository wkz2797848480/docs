---
标题: 遥测与版本检查
摘要: TimescaleDB 收集哪些遥测数据以及如何禁用遥测
产品: [自托管]
关键词: [设置，遥测]
---

# 遥测和版本检查

TimescaleDB收集匿名使用数据，以帮助我们更好地了解和协助我们的用户。它还帮助我们提供一些服务，例如自动版本检查。您的隐私对我们来说至关重要，因此我们不收集任何个人身份识别信息。特别是，`UUID`（用户ID）字段不包含任何识别信息，而是由适当种子化的随机数生成器随机生成。

这是一个特定部署所发送的JSON数据文件的示例：

<Collapsible heading="示例JSON遥测数据文件" defaultExpanded={false}>

```json
{
  "db_uuid": "860c2be4-59a3-43b5-b895-5d9e0dd44551",
  "license": {
    "edition": "community"
  },
  "os_name": "Linux",
  "relations": {
    "views": {
      "num_relations": 0
    },
    "tables": {
      "heap_size": 32768,
      "toast_size": 16384,
      "indexes_size": 98304,
      "num_relations": 4,
      "num_reltuples": 12
    },
    "hypertables": {
      "heap_size": 3522560,
      "toast_size": 23379968,
      "compression": {
        "compressed_heap_size": 3522560,
        "compressed_row_count": 4392,
        "compressed_toast_size": 20365312,
        "num_compressed_chunks": 366,
        "uncompressed_heap_size": 41951232,
        "uncompressed_row_count": 421368,
        "compressed_indexes_size": 11993088,
        "uncompressed_toast_size": 2998272,
        "uncompressed_indexes_size": 42696704,
        "num_compressed_hypertables": 1
      },
      "indexes_size": 18022400,
      "num_children": 366,
      "num_relations": 2,
      "num_reltuples": 421368
    },
    "materialized_views": {
      "heap_size": 0,
      "toast_size": 0,
      "indexes_size": 0,
      "num_relations": 0,
      "num_reltuples": 0
    },
    "partitioned_tables": {
      "heap_size": 0,
      "toast_size": 0,
      "indexes_size": 0,
      "num_children": 0,
      "num_relations": 0,
      "num_reltuples": 0
    },
    "continuous_aggregates": {
      "heap_size": 122404864,
      "toast_size": 6225920,
      "compression": {
        "compressed_heap_size": 0,
        "compressed_row_count": 0,
        "num_compressed_caggs": 0,
        "compressed_toast_size": 0,
        "num_compressed_chunks": 0,
        "uncompressed_heap_size": 0,
        "uncompressed_row_count": 0,
        "compressed_indexes_size": 0,
        "uncompressed_toast_size": 0,
        "uncompressed_indexes_size": 0
      },
      "indexes_size": 165044224,
      "num_children": 760,
      "num_relations": 24,
      "num_reltuples": 914704,
      "num_caggs_on_distributed_hypertables": 0,
      "num_caggs_using_real_time_aggregation": 24
    },
    "distributed_hypertables_data_node": {
      "heap_size": 0,
      "toast_size": 0,
      "compression": {
        "compressed_heap_size": 0,
        "compressed_row_count": 0,
        "compressed_toast_size": 0,
        "num_compressed_chunks": 0,
        "uncompressed_heap_size": 0,
        "uncompressed_row_count": 0,
        "compressed_indexes_size": 0,
        "uncompressed_toast_size": 0,
        "uncompressed_indexes_size": 0,
        "num_compressed_hypertables": 0
      },
      "indexes_size": 0,
      "num_children": 0,
      "num_relations": 0,
      "num_reltuples": 0
    },
    "distributed_hypertables_access_node": {
      "heap_size": 0,
      "toast_size": 0,
      "compression": {
        "compressed_heap_size": 0,
        "compressed_row_count": 0,
        "compressed_toast_size": 0,
        "num_compressed_chunks": 0,
        "uncompressed_heap_size": 0,
        "uncompressed_row_count": 0,
        "compressed_indexes_size": 0,
        "uncompressed_toast_size": 0,
        "uncompressed_indexes_size": 0,
        "num_compressed_hypertables": 0
      },
      "indexes_size": 0,
      "num_children": 0,
      "num_relations": 0,
      "num_reltuples": 0,
      "num_replica_chunks": 0,
      "num_replicated_distributed_hypertables": 0
    }
  },
  "os_release": "5.10.47-linuxkit",
  "os_version": "#1 SMP Sat Jul 3 21:51:47 UTC 2021",
  "data_volume": 381903727,
  "db_metadata": {},
  "build_os_name": "Linux",
  "functions_used": {
    "pg_catalog.int8(integer)": 8,
    "pg_catalog.count(pg_catalog.\"any\")": 20,
    "pg_catalog.int4eq(integer,integer)": 7,
    "pg_catalog.textcat(pg_catalog.text,pg_catalog.text)": 10,
    "pg_catalog.chareq(pg_catalog.\"char\",pg_catalog.\"char\")": 6,
  },
  "install_method": "docker",
  "installed_time": "2022-02-17T19:55:14+00",
  "os_name_pretty": "Alpine Linux v3.15",
  "last_tuned_time": "2022-02-17T19:55:14Z",
  "build_os_version": "5.11.0-1028-azure",
  "exported_db_uuid": "5730161f-0d18-42fb-a800-45df33494c21",
  "telemetry_version": 2,
  "build_architecture": "x86_64",
  "distributed_member": "none",
  "last_tuned_version": "0.12.0",
  "postgresql_version": "12.10",
  "related_extensions": {
    "postgis": false,
    "pg_prometheus": false,
    "timescale_analytics": false,
    "timescaledb_toolkit": false
  },
  "timescaledb_version": "2.6.0",
  "num_reorder_policies": 0,
  "num_retention_policies": 0,
  "num_compression_policies": 1,
  "num_user_defined_actions": 1,
  "build_architecture_bit_size": 64,
  "num_continuous_aggs_policies": 24
}
```

</Collapsible>

如果您想查看确切发送的JSON数据文件，请使用[`get_telemetry_report`][get_telemetry_report] API调用。

<Highlight type="note">
如果您使用的是TimescaleDB的开源或社区版本，遥测报告会有所不同。对于这些版本，报告中包括一个`edition`字段，其值为`apache_only`或`community`。
</Highlight>

## 更改遥测报告中包含的内容

如果您想调整哪些元数据包含或从遥测报告中排除，可以在`_timescaledb_catalog.metadata`表中进行。将`include_in_telemetry`设置为`true`的元数据，以及`timescaledb_telemetry.cloud`的值，将包含在遥测报告中。

## 版本检查

遥测报告会定期在后台发送。作为对遥测报告的响应，数据库会收到可供安装的最新版本的TimescaleDB。这个版本会记录在您的服务器日志中，以及任何适用的过时版本警告。您不必立即更新到最新版本，但我们强烈建议您这样做，以利用性能改进和错误修复。

## 禁用遥测

我们强烈建议您保持遥测功能启用，因为它为您提供了有用的功能，并有助于持续改进Timescale。但是，如果需要，您可以为特定数据库或整个
实例关闭遥测。

<Highlight type="important">
如果您关闭遥测，版本检查功能也会被关闭。
</Highlight>

<Procedure>

### 禁用遥测

1.  打开您的PostgreSQL配置文件，并找到`timescaledb.telemetry_level`参数。有关定位和打开文件的说明，请参见[PostgreSQL配置文件][postgres-config]。
2.  将参数设置更改为`off`：

    ```yaml
    timescaledb.telemetry_level=off
    ```

3.  重新加载配置文件：

    ```bash
    pg_ctl
    ```

4.  或者，您可以在`psql`提示符下作为root用户使用此命令：

    ```sql
    ALTER [SYSTEM | DATABASE | USER] { *db_name* | *role_specification* } SET timescaledb.telemetry_level=off
    ```

    此命令会禁用指定系统、数据库或用户的遥测功能。

</Procedure>

<Procedure>

### 启用遥测

1.  打开您的PostgreSQL配置文件，并找到`timescaledb.telemetry_level`参数。有关定位和打开文件的说明，请参见[PostgreSQL配置文件][postgres-config]。
2.  将参数设置更改为`basic`：

    ```yaml
    timescaledb.telemetry_level=basic
    ```

3.  重新加载配置文件：

    ```bash
    pg_ctl
    ```

4.  或者，您可以在`psql`提示符下作为root用户使用此命令：

    ```sql
    ALTER [SYSTEM | DATABASE | USER] { *db_name* | *role_specification* } SET timescaledb.telemetry_level=basic
    ```

    此命令会启用指定系统、数据库或用户的遥测功能。

</Procedure>

[get_telemetry_report]: /api/:currentVersion:/administration/#get_telemetry_report
[postgres-config]: /self-hosted/:currentVersion:/configuration/postgres-config

