---
标题: 将数据从 InfluxDB 迁移至 Timescale
摘要: 使用 Outflux 工具将数据迁移至 Timescale
产品: [自托管]
关键词: [数据迁移，InfluxDB]
标签: [导入，Outflux]
---

# 从 InfluxDB 迁移数据到 Timescale

您可以使用 Outflux 工具将数据从 InfluxDB 迁移到 Timescale。
[Outflux][outflux] 是由 Timescale 构建的开源工具，用于快速、无缝的迁移。
它直接将导出的数据管道传输到 Timescale，并管理模式发现、验证和创建。

<Highlight type="important">

Outflux 适用于 InfluxDB 的早期版本。它不适用于 InfluxDB 版本 2 及更高版本。

</Highlight>

## 前提条件

开始之前，请确保您有：

*   一个运行中的 InfluxDB 实例以及连接到它的方法。
*   一个[安装好的 Timescale][install]实例以及连接到它的方法。
*   InfluxDB 实例中的数据。

## 程序

从 Outflux 导入数据，请按照以下程序操作：

1.  [安装 Outflux][install-outflux]
2.  [发现、验证并转移模式][discover-validate-and-transfer-schema]到 Timescale（可选）
3.  [将数据迁移到 Timescale][migrate-data-to-timescale]

## 安装 Outflux

从 GitHub 仓库安装 Outflux。有适用于 Linux、Windows 和 MacOS 的构建版本。

<Procedure>

### 安装 Outflux

1.  转到 Outflux 仓库的[发布部分][outflux-releases]。
2.  下载适用于您的平台的最新压缩 tarball。
3.  将其提取到您喜欢的地点。

<Highlight type="note">

如果您更喜欢从源代码构建 Outflux，请参见[Outflux README][outflux-readme]中的说明。

</Highlight>

</Procedure>

要从安装目录获取 Outflux 的帮助，运行 `./outflux --help`。

## 发现、验证并转移模式

Outflux 可以：

*   发现 InfluxDB 测量的模式
*   验证是否存在可以容纳传输数据的 Timescale 表
*   如果不存在有效表，则创建一个新表以满足模式要求

<Highlight type="note">

Outflux 的 `migrate` 命令将模式转移和数据迁移合并为一步。
有关更多信息，请参见[迁移数据到 Timescale][migrate-data-to-timescale]部分。
如果您想独立于数据迁移来验证和转移您的模式，请使用此部分。

</Highlight>

要从 InfluxDB 到 Timescale 传输您的模式，运行 `outflux schema-transfer`：

```bash
outflux schema-transfer <DATABASE_NAME> <INFLUX_MEASUREMENT_NAME> \
--input-server=http://localhost:8086 \
--output-conn="dbname=tsdb user=tsdbadmin"
```

要传输数据库中的所有测量，请省略测量名称参数。

<Highlight type="note">

此示例使用 `postgres` 用户和数据库连接到 Timescale 数据库。
有关其他连接选项和配置，请参见[Outflux Github 仓库][outflux-gitbuh]。

</Highlight>

### 模式转移选项

Outflux 的 `schema-transfer` 可以使用 4 种模式策略之一：

*   `ValidateOnly`：检查是否安装了 Timescale 以及指定的数据库是否有正确列的适当分区的超表，但不执行修改
*   `CreateIfMissing`：运行与 `ValidateOnly` 相同的检查，并创建并正确分区任何缺失的超表
*   `DropAndCreate`：删除任何与测量名称相同的现有表，并创建一个新的超表并正确分区
*   `DropCascadeAndCreate`：执行与 `DropAndCreate` 相同的操作，并在存在与测量名称相同的现有表时执行级联表删除

您可以通过在 `schema-transfer` 命令中传递 `--schema-strategy` 选项的值来指定您的模式策略。默认策略是 `CreateIfMissing`。

默认情况下，InfluxDB 中的每个标签和字段都被视为 Timescale 表中的单独列。
要将标签和字段作为单个 JSONB 列传输，使用 `--tags-as-json` 标志。

## 将数据迁移到 Timescale

使用 `migrate` 命令一次性传输您的模式并迁移您的数据。

例如，运行：

```bash
outflux migrate <DATABASE_NAME> <INFLUX_MEASUREMENT_NAME> \
--input-server=http://localhost:8086 \
--output-conn="dbname=tsdb user=tsdbadmin"
```

模式策略和连接选项与 `schema-transfer` 相同。
有关更多信息，请参见[发现、验证并转移模式][discover-validate-and-transfer-schema]。

此外，`outflux migrate` 还接受以下标志：

*   `--limit`：传递一个数字 `N` 到 `--limit` 以仅导出前 `N` 行，按时间排序。
*   `--from` 和 `--to`：传递一个时间戳到 `--from` 或 `--to` 以指定要迁移的数据的时间窗口。
*   `chunk-size`：更改传输的数据块大小。数据以默认大小 15,000 的块从 InfluxDB 服务器拉取。
*   `batch-size`：更改插入批处理中的行数。数据默认以 8000 行为一批插入 Timescale。

有关更多标志，请参见 [Github 上的 `outflux migrate` 文档][outflux-migrate]。
或者，查看命令行帮助：

```bash
outflux migrate --help
```

[influx-cmd]: https://docs.influxdata.com/influxdb/v1.7/tools/shell/ 
[install]: /getting-started/:currentVersion:/
[outflux-migrate]: https://github.com/timescale/outflux#migrate 
[outflux-releases]: https://github.com/timescale/outflux/releases 
[outflux]: https://github.com/timescale/outflux 
[install-outflux]: /self-hosted/:currentVersion:/migration/migrate-influxdb/#install-outflux
[discover-validate-and-transfer-schema]: /self-hosted/:currentVersion:/migration/migrate-influxdb/#discover-validate-and-transfer-schema
[migrate-data-to-timescale]: /self-hosted/:currentVersion:/migration/migrate-influxdb/#migrate-data-to-timescale
[outflux-gitbuh]: https://github.com/timescale/outflux#connection 
[outflux-readme]: https://github.com/timescale/outflux/blob/master/README.md
