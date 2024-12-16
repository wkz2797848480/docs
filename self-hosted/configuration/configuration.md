---
标题: 配置 TimescaleDB
摘要: 如何配置 TimescaleDB 实例
产品: [自托管]
关键词: [配置]
标签: [设置]
---

# 配置TimescaleDB

TimescaleDB使用默认的PostgreSQL服务器配置设置。然而，我们发现这些设置通常过于保守，并且在使用资源更多的大型服务器（CPU、内存、磁盘等）时可能会受到限制。调整这些设置，无论是使用我们的`timescaledb-tune`工具[自动调整][tstune]还是手动编辑您的机器上的`postgresql.conf`文件，都可以提高性能。

<Highlight type="tip">
您可以通过在PostgreSQL客户端（例如`psql`）中运行`SHOW config_file;`来确定`postgresql.conf`的位置。
</Highlight>

此外，其他TimescaleDB特定的设置也可以通过`postgresql.conf`文件进行修改，如[TimescaleDB设置][ts-settings]部分所述。

## 使用`timescaledb-tune`

为了简化配置过程，请使用[`timescaledb-tune`][tstune]，它根据系统自动设置最常见的参数到适当的值，考虑内存、CPU和PostgreSQL版本。`timescaledb-tune`与二进制版本一起打包作为依赖项，因此如果您安装了二进制版本（包括Docker），您应该可以使用该工具。或者，使用标准的Go环境，您也可以`go get`仓库来安装它。

`timescaledb-tune`读取系统的`postgresql.conf`文件，并提供交互式建议来更新您的设置：

```bash
使用此路径的postgresql.conf文件：
/usr/local/var/postgres/postgresql.conf

这是否正确？[(y)es/(n)o]: y
将备份写入：
/var/folders/cr/zpgdkv194vz1g5smxl_5tggm0000gn/T/timescaledb_tune.backup201901071520

需要更新shared_preload_libraries
当前：
#shared_preload_libraries = 'timescaledb'
推荐：
shared_preload_libraries = 'timescaledb'
这样可以吗？[(y)es/(n)o]: y
成功：shared_preload_libraries将被更新

调整内存/并行性/WAL等设置？[(y)es/(n)o]: y
基于8.00 GB可用内存和4个CPU为PostgreSQL 11推荐的设置

内存设置推荐
当前：
shared_buffers = 128MB
#effective_cache_size = 4GB
#maintenance_work_mem = 64MB
#work_mem = 4MB
推荐：
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 1GB
work_mem = 26214kB
这样可以吗？[(y)es/(s)kip/(q)uit]:
```

然后这些更改将被写入您的`postgresql.conf`文件，并在下次（重新）启动时生效。如果您是在全新的实例上开始，并且不觉得需要批准每组更改，您也可以自动接受并将建议附加到您的`postgresql.conf`文件末尾，如下所示：

```bash
timescaledb-tune --quiet --yes --dry-run >> /path/to/postgresql.conf
```

## PostgreSQL配置和调整

如果您更倾向于自己调整设置，或者对`timescaledb-tune`提出的建议感到好奇，那么请查看这些。然而，`timescaledb-tune`并没有涵盖所有您需要调整的设置。

### 内存设置

<Highlight type="tip">
`timescaledb-tune`处理所有这些设置。
</Highlight>
需要调整`shared_buffers`、`effective_cache_size`、`work_mem`和`maintenance_work_mem`设置以匹配机器的可用内存。从[PgTune][pgtune]网站获取配置值（建议的数据库类型：数据仓库）。您还应该根据PgTune给出的设置调整`max_connections`设置，因为`max_connections`和内存设置之间存在联系。PgTune的其他设置也可能有所帮助。

### 工作进程设置

<Highlight type="tip">
`timescaledb-tune`处理所有这些设置。
</Highlight>
PostgreSQL利用工作进程池来提供支持实时查询和后台作业所需的工作进程。如果您没有配置这些设置，您可能会发现查询和后台作业的性能下降。

使用`timescaledb.max_background_workers`设置配置TimescaleDB后台工作进程。您应该将此设置配置为您的数据库总数和您希望在任何给定时间运行的并发后台工作进程的总数之和。您需要为每个数据库分配一个后台工作进程来运行轻量级调度器来安排作业。除此之外，您在这里分配的任何额外工作进程在需要时运行后台作业。

对于较大的查询，PostgreSQL在有可用的并行工作进程时会自动使用它们。要配置这个，请使用`max_parallel_workers`设置。增加此设置可以提高大型查询的查询性能。较小的查询可能不会触发并行工作进程。默认情况下，此设置对应于可用的CPU数量。使用`--cpus`标志或`TS_TUNE_NUM_CPUS` docker环境变量来更改它。

最后，您必须配置`max_worker_processes`至少为`timescaledb.max_background_workers`和`max_parallel_workers`的总和。`max_worker_processes`是后台和并行工作进程（以及一些内置的PostgreSQL工作进程）可用的工作进程池的总数。

默认情况下，`timescaledb-tune`将`timescaledb.max_background_workers`设置为16。要更改此设置，请使用`--max-bg-workers`标志或`TS_TUNE_MAX_BG_WORKERS` docker环境变量。`max_worker_processes`设置也会自动调整。

### 磁盘写入设置

为了提高写入吞吐量，有[多个设置][async-commit]可以调整PostgreSQL写数据到磁盘的行为。在测试中，使用默认或最安全设置的性能表现良好。如果您想要获得一些额外的性能提升，可以设置`synchronous_commit = 'off'`([PostgreSQL文档][synchronous-commit]）。请注意，以这种方式禁用`synchronous_commit`时，操作系统或数据库崩溃可能会导致一些最近看似已提交的事务丢失。我们强烈不鼓励更改`fsync`设置。

### 锁设置

TimescaleDB在扩展时间序列工作负载时，严重依赖于表分区，这对[锁管理][lock-management]有影响。一个超级表在查询期间需要获取许多块（子表）上的锁，这可能会耗尽默认允许持有的锁数量限制。这可能导致出现以下警告：

```sql
psql: FATAL:  out of shared memory
HINT:  You might need to increase max_locks_per_transaction.
```

为了避免这个问题，有必要将`max_locks_per_transaction`设置从默认值（通常是64）增加。由于更改此参数需要重启数据库，建议估计一个好设置，同时也允许一些增长。对于大多数用例，我们推荐以下设置：

```
max_locks_per_transaction = 2 * num_chunks / max_connections
```

其中`num_chunks`是您期望在一个超级表中拥有的最大块数，`max_connections`是为PostgreSQL配置的连接数。这考虑到如果需要在查询中访问所有块，超级表查询使用的锁数量大致等于超级表中的块数，或者如果查询使用索引，则为该数量的两倍。

您可以使用[`timescaledb_information.hypertables`][timescaledb_information-hypertables]视图查看当前有多少块。更改此参数需要重启数据库，因此请确保选择一个较大的数字以允许一些增长。有关锁管理的更多信息，请参见[PostgreSQL文档][lock-management]。

## TimescaleDB配置和调优

就像您可以在PostgreSQL中调整设置一样，TimescaleDB也提供了许多配置设置，这些设置可能对您的特定安装和性能需求有用。这些设置也可以在`postgresql.conf`文件中设置，或作为启动PostgreSQL时的命令行参数。

### 策略

#### `timescaledb.max_background_workers (int)`

分配给TimescaleDB的最大后台工作进程数。设置为至少1 + Postgres实例中的数据库数量以使用后台工作进程。默认值为8。

### 分布式超级表

#### `timescaledb.hypertable_distributed_default (enum)`

设置默认策略，为`create_hypertable()`命令创建本地或分布式超级表，当未提供`distributed`参数时。支持的值为`auto`、`local`或`distributed`。

#### `timescaledb.hypertable_replication_factor_default (int)`

全局默认副本因子值，用于在未提供`replication_factor`参数时与超级表一起使用。默认为1。

#### `timescaledb.enable_2pc (bool)`

为分布式超级表启用两阶段提交。如果禁用，则改用一阶段提交，这更快但可能导致数据不一致。默认启用。

#### `timescaledb.enable_per_data_node_queries (bool)`

如果启用，TimescaleDB将同一超级表所属的不同块合并为每个数据节点的单个查询。默认启用。

#### `timescaledb.max_insert_batch_size (int)`

作为访问节点时，TimescaleDB将插入的元组批次跨多个数据节点分割。它每个数据节点批次多达`max_insert_batch_size`元组后才刷新。设置为0禁用批次处理，恢复为逐个元组插入。默认值为1000。

#### `timescaledb.enable_connection_binary_data (bool)`

启用集群节点间交换数据的二进制格式。默认启用。

#### `timescaledb.enable_client_ddl_on_data_nodes (bool)`

启用客户端在数据节点上执行DDL操作，并不仅限于访问节点执行DDL操作。默认禁用。

#### `timescaledb.enable_async_append (bool)`

启用优化，跨数据节点异步运行远程查询。默认启用。

#### `timescaledb.enable_remote_explain (bool)`

启用从远程节点获取和显示`EXPLAIN`输出。这需要将查询发送到数据节点，因此可能受到网络连接和数据节点可用性的影响。默认禁用。

#### `timescaledb.remote_data_fetcher (enum)`

根据您计划运行的查询类型选择数据提取器类型，可以是`rowbyrow`或`cursor`。默认为`rowbyrow`。

#### `timescaledb.ssl_dir (string)`

指定连接到使用证书认证的数据节点时搜索用户证书和密钥的路径。默认为PostgreSQL数据目录下的`timescaledb/certs`。

#### `timescaledb.passfile (string)`

指定存储密码的文件名，以及在连接到使用密码认证的数据节点时使用。

### 管理

#### `timescaledb.restoring (bool)`

设置TimescaleDB为恢复模式。默认禁用。

#### `timescaledb.license (string)`

TimescaleDB许可证类型。确定启用哪些功能。该变量可以设置为`timescale`或`apache`。默认为`timescale`。

#### `timescaledb.telemetry_level (enum)`

遥测设置级别。用于确定发送哪些遥测数据。可以设置为`off`或`basic`。默认为`basic`。

#### `timescaledb.last_tuned (string)`

记录`timescaledb-tune`上次运行的时间。

#### `timescaledb.last_tuned_version (string)`

记录上次运行时使用的`timescaledb-tune`版本。

## 使用Docker更改配置

在[Docker容器][docker]中运行TimescaleDB时，有两种方法可以修改您的PostgreSQL配置。以下示例中，我们在名为`timescaledb`的Docker容器中将数据库实例的预写日志（WAL）大小从1GB修改为2GB。

#### 在Docker内修改postgres.conf

1.  打开Docker中的shell以更改运行中的容器的配置。

```bash
docker start timescaledb
docker exec -i -t timescaledb /bin/bash
```

1.  编辑然后保存配置文件，修改所需配置参数的设置（例如，`max_wal_size`）。

```bash
vi /var/lib/postgresql/data/postgresql.conf
```

1.  重启容器以重新加载配置。

```bash
docker restart timescaledb
```

1.  测试更改是否有效。

```bash
docker exec -it timescaledb psql -U postgres

postgres=# show max_wal_size;
 max_wal_size
 -------------
 2GB
```

#### 将配置参数作为启动选项指定

或者，可以通过`docker run`命令的`-c`选项传入一个或多个参数，如下所示。

```bash
docker run -i -t timescale/timescaledb:latest-pg10 postgres -cmax_wal_size=2GB
```

有关在启动时传入参数的更多示例，请参阅我们关于使用WAL-E进行增量备份的[讨论][wale]。

[async-commit]: https://www.postgresql.org/docs/current/static/wal-async-commit.html 
[chunks_detailed_size]: /api/:currentVersion:/hypertable/chunks_detailed_size
[docker]: /self-hosted/latest/install/installation-docker/
[lock-management]: https://www.postgresql.org/docs/current/static/runtime-config-locks.html 
[pgtune]: http://pgtune.leopard.in.ua/ 
[synchronous-commit]: https://www.postgresql.org/docs/current/static/runtime-config-wal.html#GUC-SYNCHRONOUS-COMMIT 
[ts-settings]: /self-hosted/:currentVersion:/configuration/timescaledb-config/
[tstune]: https://github.com/timescale/timescaledb-tune 
[wale]: /self-hosted/:currentVersion:/backup-and-restore/docker-and-wale/

[async-commit]: https://www.postgresql.org/docs/current/static/wal-async-commit.html 
[timescaledb_information-hypertables]: /api/:currentVersion:/informational-views/hypertables
[docker-conf]: /self-hosted/:currentVersion:/configuration/docker-config
[lock-management]: https://www.postgresql.org/docs/current/static/runtime-config-locks.html 
[pgtune]: http://pgtune.leopard.in.ua/ 
[postgresql-conf]: /self-hosted/:currentVersion:/configuration/postgres-config
[tstune-conf]: /self-hosted/:currentVersion:/configuration/timescaledb-tune



