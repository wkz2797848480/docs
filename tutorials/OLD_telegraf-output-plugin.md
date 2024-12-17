---
标题: 使用用于 Telegraf 的 PostgreSQL 和 TimescaleDB 输出插件收集指标
摘要: 使用 Telegraf（已弃用）收集指标。
产品: [云服务，管理服务技术（MST），自托管]
关键词: [指标，监控，Telegraf]
---

# 使用Telegraf的PostgreSQL和TimescaleDB输出插件收集指标

<Highlight type="deprecation">
这一节描述了TimescaleDB上一个已弃用的特性。我们强烈建议您不要在生产环境中使用此特性。有关一些替代方案的建议，请参阅此[Timescale论坛帖子](https://www.timescale.com/forum/t/telegraf-plugin/118)。
</Highlight>

Telegraf从多种输入源收集指标，并将它们写入多种输出源。它对于数据的收集和输出都是插件驱动的，因此它是可扩展的。它用Go语言编写，这意味着它是一个编译后的独立二进制文件，可以在任何系统上运行，无需外部依赖或包管理工具。

Telegraf是一个开源工具。它包含200多个插件，用于收集和写入不同类型数据，这些数据由处理这些数据的人员编写。Timescale提供了包含插件的Telegraf的可下载二进制文件。本教程通过几个示例介绍了如何使用Telegraf的PostgreSQL和TimescaleDB输出插件。

## 安装

开始之前，你需要[安装TimescaleDB][getting-started]以及连接到它的方法。

### 设置Telegraf

Telegraf是用Go语言编写的，该工具的当前构建过程配置为生成一个独立的二进制文件。因此，所有不同插件的代码必须是该二进制文件的一部分。Timescale提供了一个非官方的Telegraf版本1.13.0的构建版本，包含插件，你可以从以下地址下载：

<!--- 这些链接不再有效，已删除。LKB 2023-05-10

*   Linux amd64: <Tag type="download">[deb]()</Tag> <Tag type="download">[rpm]()</Tag> <Tag type="download">[binary]()</Tag>
*   Windows amd64: <Tag type="download">[binary/exe]()</Tag>
*   MacOS amd64: <Tag type="download">[binary]()</Tag>

-->

Timescale还为你提供了以下构建版本：

*   Windows i386
*   Linux (i386, armhf, armel, arm64, static_amd64, s390x, mipsel)
*   FreeBSD (amd64, i386)

你可以通过Timescale[社区Slack][public-slack]与我们联系

下载二进制文件并将其解压到合适的位置（或安装包）后，你可以测试构建版本。你可能需要通过运行`chmod +x telegraf`使文件可执行。使用此命令检查安装的Telegraf版本：

```bash
telegraf --version
```

如果安装成功，它将显示`Telegraf 1.13.0-with-pg`。

## Telegraf配置

启动Telegraf时，你需要指定一个配置文件。配置文件设置：

*   Telegraf代理
*   收集间隔
*   抖动
*   缓冲区和批量大小等
*   添加到所有输入收集的指标的全局标签
*   启用的输出、处理器、聚合器、输入（及其各自的配置）

使用此命令生成包含PostgreSQL作为插件的示例配置文件：

```bash
telegraf --input-filter=cpu --output-filter=postgresql config > telegraf.conf
```

此命令生成一个配置文件，启用CPU输入插件，该插件采样各种CPU使用情况的指标，以及PostgreSQL输出插件。该文件还包括所有可用的输入、输出、处理器和聚合器插件，已注释，以便你可以根据需要启用它们。

### 测试配置文件

要测试你的配置，你可以将单次收集输出到`STDOUT`，如下所示：

```bash
telegraf --config telegraf.conf --test
```

此命令选择生成的配置文件，仅启用CPU输入插件。输出应该像这样：

```bash
> cpu,cpu=cpu0,host=local usage_guest=0,usage_idle=78.431372,usage_iowait=0,usage_irq=0,usage_softirq=0,usage_steal=0,usage_system=11.764705,usage_user=9.803921 1558613882000000000
> cpu,cpu=cpu1,host=local usage_guest=0,usage_idle=92.156862,usage_iowait=0,usage_irq=0,usage_softirq=0,usage_steal=0,usage_system=3.921568,usage_user=3.921568 1558613882000000000
> cpu,cpu=cpu-total,host=local usage_guest=0,usage_idle=87.623762,usage_iowait=0,usage_irq=0,usage_softirq=0,usage_steal=0,usage_system=6.435643,usage_user=5.940594 1558613882000000000
```

为CPU的每个核心和总计输出一行。值以`key=value`对的形式呈现，时间戳在行尾。
当写入STDOUT时，你可以通过一个空格区分*标签*（索引字段（`cpu`，`host`））和值*字段*（如`usage_quest`或`usage_user`）。这种区别存在是因为不同的字段有不同的配置选项。

### 配置PostgreSQL输出插件

你生成的`telegraf.conf`文件有一个部分（大约在第80行）以

```txt
################################################
#                OUTPUT PLUGINS                #
################################################
```

在此标题下，显示了PostgreSQL输出插件的默认配置。它看起来像这样：

```txt
[[outputs.postgresql]]
  ## 通过url指定地址，匹配：
  ##   postgres://[pqgotest[:password]]@localhost[/dbname]\
  ##       ?sslmode=[disable|verify-ca|verify-full]
  ## 或一个简单的字符串：
  ##   host=localhost user=pqotest password=... sslmode=... dbname=app_production
  ##
  ## 所有连接参数都是可选的。还支持PG环境变量
  ##  例如。PGPASSWORD, PGHOST, PGUSER, PGDATABASE
  ## 所有支持的变量在这里：https://www.postgresql.org/docs/current/libpq-envars.html 
  ##
  ## 没有dbname参数，驱动程序默认使用与用户同名的数据库。这个dbname只是用于与服务器建立连接，并不限制我们尝试获取指标的数据库。
  ##
  connection = "host=localhost user=postgres sslmode=verify-full"

  ## 将标签存储为指标表中的外键。默认为false。
  # tags_as_foreignkeys = false

  ## 生成表使用的模板
  ## 可用变量：
  ##   {TABLE} - 表名作为标识符
  ##   {TABLELITERAL} - 表名作为字符串字面量
  ##   {COLUMNS} - 列定义
  ##   {KEY_COLUMNS} - 键列（时间+标签）的逗号分隔列表
  ## 默认模板
  # table_template = "CREATE TABLE IF NOT EXISTS {TABLE}({COLUMNS})"
  ## 例如用于timescaledb
  # table_template = "CREATE TABLE {TABLE}({COLUMNS}); SELECT create_hypertable({TABLELITERAL},'time');"

  ## 创建表的模式
  # schema = "public"

  ## 使用jsonb数据类型存储标签
  # tags_as_jsonb = false
  ## 使用jsonb数据类型存储字段
  # fields_as_jsonb = false
```

从配置中，你可以看到一些重要事项：

*   第一行启用了插件，插件特定配置在这一行后面缩进。
*   目前只配置了一个参数，`connection`。其他都被注释掉了。
*   可能的参数被注释掉了，用单个`#`。
    （`tags_as_foreignkeys`，`table_template`，`schema`，`tags_as_jsonb`，
    `fields_as_jsonb`）。
*   参数的解释被注释掉了，用`##`。

被注释掉的参数还显示了它们的默认值。

在第一个示例中，你将设置连接参数为一个适当的连接字符串，以建立与TimescaleDB或PostgreSQL实例的连接。
所有其他参数都使用它们的默认值。

### 创建超表

插件允许你配置几个参数。`table_template`参数定义了当Telegraf记录新测量值时需要运行的SQL，并且输出数据库中不存在所需表。默认情况下，使用的`table_template`是`CREATE TABLE IF NOT EXISTS {TABLE}({COLUMNS})`，其中`{TABLE}`和`{COLUMNS}`是表名和列定义的占位符。

你可以使用此命令为TimescaleDB更新`table_template`配置：

```sql
  table_template=`CREATE TABLE IF NOT EXISTS {TABLE}({COLUMNS}); SELECT create_hypertable({TABLELITERAL},'time',chunk_time_interval := INTERVAL '1 week',if_not_exists := true);`
```

这样，当创建一个新表时，它被转换为超表，每个块保存1周的间隔。使用该插件与TimescaleDB不需要其他任何操作。

## 运行Telegraf

运行Telegraf时，你只需要指定要使用的配置文件。在这个例子中，输出使用已加载的输入（`cpu`）和输出（`postgresql`）以及全局标签，以及代理从输入收集数据和刷新到输出的间隔。你可以在大约10-15秒后停止运行Telegraf：

```bash
telegraf --config telegraf.conf
2019-05-23T13:48:09Z I! Starting Telegraf 1.13.0-with-pg
2019-05-23T13:48:09Z I! Loaded inputs: cpu
2019-05-23T13:48:09Z I! Loaded outputs: postgresql
2019-05-23T13:48:09Z I! Tags enabled: host=local
2019-05-23T13:48:09Z I! [agent] Config: Interval:10s, Quiet:false, Hostname:"local", Flush Interval:10s
```

现在你可以连接到PostgreSQL实例并检查数据：

```bash
psql -U postgres -h localhost
```

CPU输入插件有一个测量值，称为`cpu`，它存储在同名的表中（默认在公共模式下）。因此，使用SQL查询`SELECT * FROM cpu`，根据你让Telegraf运行的时间，你将看到表中填充了一些值。你可以使用`SELECT cpu, avg(usage_user) FROM cpu GROUP BY cpu`找到每个CPU核心的平均使用率。输出应该像这样：

```sql
    cpu    |       avg
-----------+------------------
 cpu-total | 8.46385703620795
 cpu0      | 12.4343351351033
 cpu1      | 4.88380203380203
 cpu2      | 12.2718724052057
 cpu3      | 4.26716970050303
```

### 添加新标签或字段

你的Telegraf配置可以随时更改。输入插件可以重新配置以产生不同的数据，或者你可能决定使用不同的标签索引你的数据。SQL插件可以动态更新创建的表，随着新列的出现。之前的配置没有指定全局标签，除了`host`标签。现在你可以通过打开任何文本编辑器并更新`[global_tags]`部分（大约在第18行）来添加一个新的全局标签：

```txt
[global_tags]
  location="New York"
```

这样，所有使用此配置运行的Telegraf实例收集的指标都被标记为`location="New York"`。如果你再次运行Telegraf，收集TimescaleDB中的指标，使用此命令：

```bash
telegraf --config telegraf.conf
```

过一会儿，你可以检查数据库中的`cpu`表，像这样：

```sql
psql> \dS cpu
\dS cpu;
Table "public.cpu"
      Column      |           Type
------------------+--------------------------
 time             | timestamp with time zone
 cpu              | text
 host             | text
 usage_steal      | double precision
 usage_iowait     | double precision
 usage_guest      | double precision
 usage_idle       | double precision
 usage_softirq    | double precision
 usage_system     | double precision
 usage_user       | double precision
 usage_irq        | double precision
 location         | text
 ```

你可以看到`location`列被添加了，并且它包含所有行的`New York`。

### 创建单独的元数据表以存储标签

插件允许你选择插入单独表中的标签集，然后使用外键在测量表中引用。在单独的表中存储标签可以节省高基数标签集的空间，并允许某些查询更有效地编写。要启用此更改，你需要取消注释插件配置中的`tags_as_foreignkeys`参数（大约在`telegraf.conf`的第103行）并将其设置为true：

```txt
## 将标签存储为指标表中的外键。默认为false。
tags_as_foreignkeys = true
```

为了更好地可视化结果，你可以从数据库中删除现有的`cpu`表：

```sql
psql> DROP TABLE cpu;
```

现在你可以再次启动Telegraf，这次配置更改为将标签写入单独的表：

```bash
telegraf --config telegraf.conf
```

你可以在20-30秒后将其关闭，并检查数据库中的`cpu`表：

```sql
psql> \dS cpu
\dS cpu
Table "public.cpu"
      Column      |           Type
------------------+--------------------------
 time             | timestamp with time zone
 tag_id           | integer
 usage_irq        | double precision
 usage_softirq    | double precision
 usage_system     | double precision
 usage_iowait     | double precision
 usage_guest      | double precision
 usage_user       | double precision
 usage_idle       | double precision
 usage_steal      | double precision
```

现在`cpu`、`host`和`location`列不在了，相反有一个`tag_id`列。标签集存储在名为`cpu_tag`的单独表中：

```sql
 psql> SELECT * FROM cpu_tag;
 tag_id |  host |    cpu    |  location
--------+-------+-----------+----------
      1 | local | cpu-total | New York
      2 | local | cpu0      | New York
      3 | local | cpu1      | New York
```

### JSONB列用于标签和字段

标签和字段可以作为JSONB列存储在数据库中。你需要取消注释`telegraf.conf`中的`tags_as_jsonb`或`fields_as_jsonb`参数（大约在第120行）并将它们设置为`true`。在这个例子中，字段被存储为单独的列，但标签被存储为JSON：

```txt
## 使用jsonb数据类型存储标签
tags_as_jsonb = true
## 使用jsonb数据类型存储字段
fields_as_jsonb = false
```

为了更好地可视化结果，从数据库中删除现有的`cpu_tag`表：

```sql
psql> DROP TABLE cpu_tag;
```

再次启动Telegraf，然后20-30秒后关闭。然后检查数据库中的`cpu_tag`表：

```bash
telegraf --config telegraf.conf
```

```sql
 psql> SELECT * FROM cpu_tag;
 tag_id |                                       tags
--------+-----------------------------------------------------------------------------------
      1 | {"cpu": "cpu-total", "host": "local", "location": "New York"}
      2 | {"cpu": "cpu0", "host": "local", "location": "New York"}
      3 | {"cpu": "cpu1", "host": "local", "location": "New York"}
```

现在你不是有三个文本列，而是有一个JSONB列。

## 后续步骤

当你开始在TimescaleDB中插入数据后，你可以开始熟悉[API参考][api]。

此外，还有几个其他的[tutorials][]供你探索，当你习惯使用TimescaleDB时。

[api]: /api/:currentVersion:/
[getting-started]: /getting-started/latest/
[public-slack]: https://slack.timescale.com/ 
[tutorials]: /tutorials/:currentVersion:/

