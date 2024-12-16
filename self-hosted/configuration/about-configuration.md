---
标题: 关于 TimescaleDB 中的配置
摘要: 关于 TimescaleDB 的配置情况
产品: [自托管]
关键词: [配置，内存，工作线程，设置]
---


# 关于TimescaleDB中的配置

默认情况下，TimescaleDB使用默认的PostgreSQL服务器配置设置。然而，在某些情况下，这些设置并不合适，特别是当您拥有较大的服务器，使用更多的硬件资源，如CPU、内存和存储时。本节解释了一些您最有可能需要调整的设置。

这些设置中有些是PostgreSQL设置，有些是TimescaleDB特定的设置。对于大多数更改，您可以使用[tuning tool][tstune-conf]来调整您的配置。对于更高级的配置设置，或者要更改`timescaledb-tune`工具未包含的设置，您可以[手动调整][postgresql-conf]`postgresql.conf`配置文件。

## 内存

设置：

*   `shared_buffers`
*   `effective_cache_size`
*   `work_mem`
*   `maintenance_work_mem`
*   `max_connections`

您可以调整这些设置以匹配机器的可用内存。为了简化操作，您可以使用[PgTune][pgtune]网站来计算要使用的设置：输入您的机器详细信息，并选择`data warehouse`数据库类型以查看建议的参数。

<Highlight type="tip">
您可以使用`timescaledb-tune`调整这些设置。
</Highlight>

## 工作进程

设置：

*   `timescaledb.max_background_workers`
*   `max_parallel_workers`
*   `max_worker_processes`

PostgreSQL使用工作进程池为实时查询和后台作业提供工作进程。如果您没有配置这些设置，您的查询和后台作业可能会运行得更慢。

TimescaleDB后台工作进程使用`timescaledb.max_background_workers`进行配置。每个数据库需要分配一个后台工作进程以安排作业。额外的工作进程根据需要运行后台作业。此设置应为您想要同时运行的数据库总数和并发后台工作进程的总数之和。默认情况下，`timescaledb-tune`将`timescaledb.max_background_workers`设置为16。您可以直接更改此设置，使用`--max-bg-workers`标志，或调整`TS_TUNE_MAX_BG_WORKERS`[Docker环境变量][docker-conf]。

TimescaleDB并行工作进程使用`max_parallel_workers`进行配置。对于较大的查询，PostgreSQL在有可用的并行工作进程时会自动使用它们。增加此设置可以提高触发使用并行工作进程的大型查询的查询性能。默认情况下，此设置对应于可用的CPU数量。您可以直接更改此参数，通过调整`--cpus`标志，或使用`TS_TUNE_NUM_CPUS`[Docker环境变量][docker-conf]。

`max_worker_processes`设置定义了后台和并行工作进程可用的工作进程池的总数，以及一小部分内置的PostgreSQL工作进程。它至少应该是`timescaledb.max_background_workers`和`max_parallel_workers`的总和。

<Highlight type="tip">
您可以使用`timescaledb-tune`调整这些设置。
</Highlight>

## 磁盘写入

设置：

*   `synchronous_commit`

默认情况下，磁盘写入是同步执行的，因此在下一个事务可以开始之前，每个事务都必须完成并发送成功消息。您可以通过设置`synchronous_commit = 'off'`将其更改为异步，以提高写入吞吐量。请注意，禁用同步提交可能会导致一些已提交的事务丢失。为了帮助降低风险，不要同时更改`fsync`设置。有关异步提交和磁盘写入速度的更多信息，请参见[PostgreSQL文档][async-commit]。

<Highlight type="tip">
您可以在`postgresql.conf`配置文件中调整这些设置。
</Highlight>

## 事务锁

设置：

*   `max_locks_per_transaction`

TimescaleDB依赖于表分区来扩展时间序列工作负载。超表在查询期间需要在许多块上获取锁，这可能会耗尽允许持有的锁的默认限制。在某些情况下，您可能会看到这样的警告：

```sql
psql: FATAL:  out of shared memory
HINT:  You might need to increase max_locks_per_transaction.
```

为了避免这个问题，您可以将`max_locks_per_transaction`设置从默认值（通常是64）增加。此参数限制了每个事务使用的对象锁的平均数量；只要所有事务的锁都能适应锁表，单个事务就可以锁定更多的对象。

对于大多数工作负载，选择的数字应等于您期望在超表中拥有的最大块数的两倍除以`max_connections`。这考虑到了如果需要在查询中访问所有块，超表查询使用的锁的数量大致等于超表中的块数，或者如果查询使用索引，则为该数量的两倍。

您可以使用[`timescaledb_information.hypertables`][timescaledb_information-hypertables]视图查看当前有多少块。更改此参数需要数据库重启，因此请确保选择一个较大的数字以允许一些增长。有关锁管理的更多信息，请参见[PostgreSQL文档][lock-management]。

<Highlight type="tip">
您可以在`postgresql.conf`配置文件中调整这些设置。
</Highlight>

[async-commit]: https://www.postgresql.org/docs/current/static/wal-async-commit.html 
[timescaledb_information-hypertables]: /api/:currentVersion:/informational-views/hypertables
[docker-conf]: /self-hosted/:currentVersion:/configuration/docker-config
[lock-management]: https://www.postgresql.org/docs/current/static/runtime-config-locks.html 
[pgtune]: http://pgtune.leopard.in.ua/ 
[postgresql-conf]: /self-hosted/:currentVersion:/configuration/postgres-config
[tstune-conf]: /self-hosted/:currentVersion:/configuration/timescaledb-tune


[async-commit]: https://www.postgresql.org/docs/current/static/wal-async-commit.html
[timescaledb_information-hypertables]: /api/:currentVersion:/informational-views/hypertables
[docker-conf]: /self-hosted/:currentVersion:/configuration/docker-config
[lock-management]: https://www.postgresql.org/docs/current/static/runtime-config-locks.html
[pgtune]: http://pgtune.leopard.in.ua/
[postgresql-conf]: /self-hosted/:currentVersion:/configuration/postgres-config
[tstune-conf]: /self-hosted/:currentVersion:/configuration/timescaledb-tune
