---
标题: 定制 TimescaleDB 仪表盘
摘要: 使用 Hasura GraphQL 和 React 构建定制的 TimescaleDB 仪表盘。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [可视化，分析，Hasura]
---

# 自定义TimescaleDB仪表板

为了帮助您理解数据库中的情况，您可以创建自己的自定义可视化和仪表板。TimescaleDB允许您使用PostgreSQL监控的全部功能为您的数据创建自定义仪表板。当然，您也可以使用其他商业工具来监控TimescaleDB，就像您可以对PostgreSQL做的那样，但自定义仪表板为您提供了最大的灵活性。

本教程向您展示如何构建一个自定义可视化，显示超表拥有的块数量、每个块的压缩状态以及数据库当前的总大小。前端使用React构建，并使用Hasura（一个GraphQL服务）连接到TimescaleDB的指标。本教程包括：

*   适用于可视化的TimescaleDB概念
*   如何查询TimescaleDB视图和函数以获取有关超表和块的详细信息
*   如何生成样本数据
*  Hasura如何帮助通过GraphQL订阅流式传输数据
*   如何构建您的React前端以可视化数据

该项目使用React，连接到[Hasura][] GraphQL API以可视化[超表块][hypertables]的[TimescaleDB][]实例。

获取TimescaleDB实例的最简单方式是使用我们的托管服务[免费试用][timescale-signup]。您也可以[免费下载TimescaleDB][timescale-install]并在本地或您自己的云基础设施中运行。

您可以从[this GitHub repo][repo-example]获取此项目的完整代码。

此项目适用于任何TimescaleDB实例，但如果您有兴趣生成样本数据使用，请使用我们的模拟IoT传感器数据教程。

## TimescaleDB如何管理时间序列数据

TimescaleDB使用[超表][]来存储时间序列数据。TimescaleDB自动将超表中的数据分区到称为块的较小子表中。这些块代表特定时间段的数据，这使得随时间查询和管理变得更加容易。例如，如果您想要查询从上午10点到11点的数据，TimescaleDB将只扫描包含该时期数据的特定块，而不是扫描整个数据库。所有与数据库的交互仍然使用SQL在超表上进行，但TimescaleDB分区数据以使大型查询更加高效。

TimescaleDB的许多功能都依赖于块，包括[连续聚合][caggs]、[数据保留][]和本地[压缩][]。本地压缩对于大型时间序列数据集特别有帮助。时间序列数据在数量和速度上可能是无情的，并且在没有专门构建的时间序列数据库的情况下，存储和查询都很困难。您可以使用TimescaleDB压缩来节省高达97%的磁盘空间，用于相同数量的数据，并且通常还可以随着时间的推移提高查询速度。

可视化超表的状态可以帮助您更好地理解压缩的工作方式，甚至可能了解不同类型的压缩效率。可视化可以帮助您逐表查看压缩结果，逐块查看。为此，TimescaleDB提供了多个视图和函数，可以查询有关您的超表和块状态的信息。尽管没有综合视图提供我们可视化所需的确切数据，但TimescaleDB提供了构建自定义SQL查询的构建块，以返回可视化当前超表和块压缩状态所需的数据。例如，此查询返回该块覆盖的名称和时间序列范围：

```sql
tsdb=> SELECT chunk_name, range_start, range_end FROM timescaledb_information.chunks LIMIT 1;
    chunk_name    |      range_start       |       range_end
------------------+------------------------+------------------------
 _hyper_2_2_chunk | 2021-04-29 00:00:00+00 | 2021-05-06 00:00:00+00
(1 row)
```

## 可视化表和块

拥有跨越巨大时间段的数据的超表可能有数千个块，因此有效地可视化它们非常重要。为了提供表的视觉视角，图像区域代表压缩前所有表数据的总大小。每个圆圈代表一个块，每个圆圈的面积代表磁盘上块的大小。

以下是这种可视化的样子：

<img class="main-content__illustration" src="https://assets.timescale.com/docs/images/custom-timescaledb-dashboards-hypertables-compression.webp" width={2500} height={1468} alt="Hypertables compression preview"/>

通过这种可视化，您可以一目了然地看到一些东西：

*   当前有多少块是这个超表的一部分
*   每个块的压缩状态
*   通过在某些块上启用压缩节省了多少空间

通过使用未压缩数据大小来表示图像的面积，您可以快速了解整体白色空间在图像中节省了多少空间。较小的黄色块是压缩的，它们的大小代表它们在更大表中的空间比例，而较大的暗色块是未压缩的，在图像中占用更多的空间。您还可以使可视化交互式，以便您可以点击一个块并手动压缩或解压缩它。

## 在TimescaleDB中创建内部视图以获取指标

为了构建可视化应用程序，我们创建了一些新的函数和视图：

*   从块中提取信息，如名称和时间范围
*   获取有关哪些块被压缩的额外详细信息
*   获取压缩统计信息并获取压缩后块的大小

### 从块中提取信息

要从块中提取信息，您可以使用TimescaleDB扩展提供的`timescaledb_information.chunks`视图。

此查询返回每个块的时间序列范围：

```sql
 SELECT hypertable_schema,
    hypertable_name,
    chunk_name,
    range_start,
    range_end
 FROM timescaledb_information.chunks LIMIT 1;
```

垂直输出的样本行以供探索：

```sql
-[ RECORD 1 ]-----+-----------------------
hypertable_schema | public
hypertable_name   | conditions
chunk_name        | _hyper_2_2_chunk
range_start       | 2021-04-29 00:00:00+00
range_end         | 2021-05-06 00:00:00+00
```

返回的数据集的块名称是唯一的，并且可以在其他查询中使用以检索有关每个块的增强详细信息。在这个例子中，块有一个`range_start`和`range_end`，跨越了一周。随着新数据插入表中，任何时间戳在2021-04-29和2021-05-06之间的数据都存储在这个特定的`conditions`表的块上。

### 获取块的压缩状态详细信息

当您知道每个块的名称和时间范围时，您需要获取有关压缩状态以及通过压缩数据节省了多少磁盘空间的更多详细信息。您可以通过使用`chunk_compression_stats`函数查询`conditions`超表来获取这些额外信息：

```sql
tsdb=> SELECT * FROM chunk_compression_stats('conditions');
-[ RECORD 1 ]------------------+----------------------
chunk_schema                   | _timescaledb_internal
chunk_name                     | _hyper_6_913_chunk
compression_status             | Compressed
before_compression_table_bytes | 204800
before_compression_index_bytes | 360448
before_compression_toast_bytes | 0
before_compression_total_bytes | 565248
after_compression_table_bytes  | 8192
after_compression_index_bytes  | 16384
after_compression_toast_bytes  | 98304
after_compression_total_bytes  | 122880
node_name                      |
-[ RECORD 2 ]------------------+----------------------
chunk_schema                   | _timescaledb_internal
chunk_name                     | _hyper_6_880_chunk
compression_status             | Uncompressed
before_compression_table_bytes |
before_compression_toast_bytes |
before_compression_total_bytes |
after_compression_table_bytes  |
after_compression_index_bytes  |
after_compression_toast_bytes  |
after_compression_total_bytes  |
node_name                      |
```

### 获取压缩统计信息和大小

当块未压缩时，此查询不显示块的大小。要获取未压缩块的大小，请使用`chunks_detailed_size`函数，并传递超表名称作为参数：

```sql
tsdb=> SELECT * FROM chunks_detailed_size('conditions');
-[ RECORD 1 ]+----------------------
chunk_schema | _timescaledb_internal
chunk_name   | _hyper_6_853_chunk
table_bytes  | 8192
index_bytes  | 40960
toast_bytes  | 98304
total_bytes  | 147456
node_name    |
```

您可以使用此函数中的`total_bytes`信息查看块是未压缩的。

### 构建TimescaleDB指标的视图

现在您知道如何收集驱动可视化所需的所有数据，是时候将它们组合在一个可以使用SQL查询的视图中了（最终，我们的应用程序将查询此视图）。

```sql
CREATE OR REPLACE VIEW chunks_with_compression AS
SELECT DISTINCT ch.chunk_name,
                ccs.chunk_schema,
                ch.hypertable_schema,
                ch.hypertable_name,
                ch.range_start,
                ch.range_end,
                COALESCE(ccs.before_compression_total_bytes, NULL, cds.total_bytes) AS before_compression_total_bytes,
                ccs.after_compression_total_bytes
FROM (
 SELECT hypertable_schema,
    hypertable_name,
    chunk_name,
    range_start,
    range_end
 FROM  timescaledb_information.chunks) AS ch
  LEFT OUTER JOIN LATERAL chunk_compression_stats(ch.hypertable_name::regclass) ccs
    ON ch.chunk_name = ccs.chunk_name
  LEFT OUTER JOIN LATERAL chunks_detailed_size(ch.hypertable_name::regclass) cds
    ON ccs.chunk_schema = cds.chunk_schema
    AND ch.chunk_name = cds.chunk_name;
```

<Highlight type="warning">
视图依赖于TimescaleDB内部。您可能需要在升级TimescaleDB扩展时删除视图，并在升级后重新创建它。
</Highlight>

要测试，请使用超表中的随机块名称来查询此视图，并检查您是否获得了所需的所有信息。您应该看到块的时间范围、超表信息以及压缩前后的大小。

在这个示例块中，`before_compression_total_bytes`是`after_compression_total_bytes`的十倍。压缩节省了超过90%的磁盘空间！

```sql
SELECT * FROM  chunks_with_compression;
...
-[ RECORD 96 ]-----------------+-----------------------
chunk_name                     | _hyper_2_37_chunk
chunk_schema                   | _timescaledb_internal
hypertable_schema              | public
hypertable_name                | conditions
range_start                    | 2021-05-27 00:00:00+00
range_end                      | 2021-06-03 00:00:00+00
before_compression_total_bytes | 90112
after_compression_total_bytes  | 8192
```



## 设置数据库

在这个例子中，我们使用的是我们的模拟IoT传感器数据教程生成的数据。这些数据产生了一个简单的模式和数据，模仿了多个IoT传感器的信息，包括时间、设备和温度。

按照教程操作后，您将拥有一个名为`conditions`的表，该表存储了示例设备随时间变化的温度数据。

使用以下命令创建表并生成一些样本数据：

```sql
CREATE TABLE conditions (
      time TIMESTAMPTZ NOT NULL,
      device INTEGER NOT NULL,
      temperature FLOAT NOT NULL,
      PRIMARY KEY(time, device)
);

SELECT * FROM create_hypertable('conditions', 'time', 'device', 3);

INSERT INTO conditions
  SELECT time, (random()*30)::int, random()*80 - 40
  FROM generate_series(TIMESTAMP '2020-01-01 00:00:00',
                       TIMESTAMP '2020-01-01 00:00:00' + INTERVAL '1 month',
             INTERVAL '1 min') AS time;
```

## 连接数据库和检索指标

当您编写后端应用程序时，需要保护数据库并仅向授权用户暴露所需的信息。Hasura GraphQL Engine通过为新的或现有的PostgreSQL数据库提供GraphQL API来实现这一点。这允许您创建权限规则并动态扩展数据库资源。

当您设置好样本数据库后，可以使用[Hasura云][hasura-cloud]连接我们想要通过GraphQL公开的资源。Hasura是一个很好的选择，因为它连接到我们的TimescaleDB数据库，并快速公开您需要的表、视图和函数。有关在Hasura上设置新数据源的更多信息，请查看他们的向导。

我们将使用两种类型的操作：

*   查询和订阅：监视特定查询并持续向客户端拉取数据更新。在这个例子中，您订阅块的元数据。
*   突变：写入数据操作的约定。在这个例子中，您将压缩和解压缩操作映射为突变。

### 查询和订阅

Hasura允许您附加任何资源并将其作为查询或订阅提供。在这个例子中，您将之前创建的`chunks_with_compression`视图映射为GraphQL资源，因此它可以被用作查询或订阅。然后，您可以将变化或突变映射为压缩和解压缩块。这张图片描述了在Hasura上跟踪SQL视图：

![在Hasura云上跟踪SQL视图](https://assets.timescale.com/docs/images/tutorials/visualizing-compression/hasura-cloud-track-view.png) 

### 突变

Hasura可以将来自表结构的自定义类型映射。要创建必要的突变，函数需要返回继承自表结构的类型。要从查询中创建表的新结构，请使用限制为0的查询调用：

#### 压缩块突变

```sql
CREATE TABLE compressed_chunk AS
SELECT compress_chunk((c.chunk_schema ||'.' ||c.chunk_name)::regclass)
FROM   timescaledb_information.chunks c
WHERE  NOT c.is_compressed limit 0;
```

Hasura需要一个函数作为突变进行跟踪。创建一个函数重新包装TimescaleDB扩展中的默认`compress_chunk`，并在压缩块的函数中返回“compressed_chunk”：

```sql
CREATE OR REPLACE FUNCTION compress_chunk_named(varchar) returns setof compressed_chunk AS $$
  SELECT compress_chunk((c.chunk_schema ||'.' ||$1)::regclass)
  FROM   timescaledb_information.chunks c
  WHERE  NOT c.is_compressed
  AND    c.chunk_name = $1 limit 1
$$ LANGUAGE SQL VOLATILE;
```

请注意，该函数添加了一个额外的`where`子句，以便它不会压缩已经压缩的块。

![在Hasura云上跟踪压缩块突变](https://assets.timescale.com/docs/images/tutorials/visualizing-compression/hasura-cloud-compress-chunk-mutation.png) 

#### 解压缩块突变

您还需要一个类似的函数进行解压缩：

```sql
CREATE OR REPLACE FUNCTION decompress_chunk_named(varchar) returns setof compressed_chunk AS $$
  SELECT decompress_chunk((c.chunk_schema ||'.' ||$1)::regclass)
  FROM   timescaledb_information.chunks c
  WHERE  c.is_compressed
  AND    c.chunk_name = $1 limit 1
$$ LANGUAGE SQL VOLATILE;
```

下一步是前往Hasura云并将数据库连接为新的数据源。在数据面板中，设置数据库的PostgreSQL URI，然后您可以将每个函数作为查询或突变进行跟踪。这是`compress_chunk_named`函数的一个示例。在我们的例子中，订阅是`chunks_with_compression`函数。您也可以将`decompress_chunk_named`和`compress_chunk_named`作为具有单个参数的GQL突变进行跟踪。

## 构建前端可视化

有关我们前端应用程序的完整代码，请查看我们的[GitHub仓库][repo-example]。前端应用程序连接到您创建的Hasura GraphQL层，然后连接到TimescaleDB数据库以检索有关块和压缩状态的信息。然后前端应用程序渲染可视化图像。

作为总结，前端：

1.  使用GraphQL订阅API
1.  创建SVG组件
1.  遍历所有块，并在前面的组件中添加圆圈
1.  样式化圆圈并添加事件以与图像交互

## 总结

TimescaleDB是一个强大的关系数据库，用于时间序列数据，带来了PostgreSQL可用的全套工具和仪表板。

在本教程中，您学会了如何从TimescaleDB内部收集超表元数据。通过GraphQL公开它，并使用React客户端获取数据。

您可以从[这个GitHub仓库][repo-example]获取这个项目的完整代码。

本教程最初是为HasuraCon 2021创建的。

[![点击这里观看视频](https://assets.timescale.com/docs/images/tutorials/visualizing-compression/hasuracon-talk-thumbnail.png)](https://hasura.io/events/hasura-con-2021/talks/visualizing-timescale-db-%20compression-status-in-real-time-with-hasura/  "使用Hasura实时观看压缩状态")

我们希望您能找到新的方法来探索您的数据，并使您的决策更智能、数据驱动。如果您有任何有趣的结果或对本教程有任何疑问，请在我们的[社区Slack频道][timescale-slack]上留言。

[Hasura]: http://hasura.io/ 
[TimescaleDB]: https://timescale.com/ 
[caggs]: /use-timescale/:currentVersion:/continuous-aggregates/
[compression]: /use-timescale/:currentVersion:/compression/
[data retention]: /use-timescale/:currentVersion:/data-retention/
[hasura-cloud]: https://cloud.hasura.io/ 
[hypertables]: /use-timescale/:currentVersion:/hypertables/
[repo-example]: https://github.com/timescale/examples/tree/master/compression-preview 
[timescale-install]: /getting-started/latest/
[timescale-signup]: http://console.cloud.timescale.com/signup 
[timescale-slack]: https://slack.timescale.com
