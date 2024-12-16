---
标题: 配置复制
摘要: 在一个或多个数据库副本上设置异步流复制
产品: [自托管]
关键词: [副本]
---

import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";

# 配置复制

本节概述了如何在一个或多个数据库副本上设置异步流复制。

<ConsiderCloud />

在开始之前，请确保您至少有两个独立的TimescaleDB实例正在运行。如果您使用Docker容器安装了TimescaleDB，请使用[PostgreSQL入口脚本][docker-postgres-scripts]来运行配置。有关更高级的示例，请参见[Timescale Helm Charts仓库][timescale-streamrep-helm]。

要在自托管的TimescaleDB上配置复制，您需要执行以下步骤：

1.  [配置主数据库][configure-primary-db]
1.  [配置复制参数][configure-params]
1.  [创建复制槽][create-replication-slots]
1.  [配置基于主机的认证参数][configure-pghba]
1.  [在副本上创建基础备份][create-base-backup]
1.  [配置复制和恢复设置][configure-replication]
1.  [验证副本是否正常工作][verify-replica]

## 配置主数据库

要配置主数据库，您需要一个PostgreSQL用户，该用户具有允许其初始化流复制的角色。这是每个副本用于从主数据库流复制的用户。

<Procedure>

### 配置主数据库

1.  在主数据库上，以具有超级用户权限的用户（例如`postgres`用户）身份，将密码加密级别设置为`scram-sha-256`：

    ```sql
    SET password_encryption = 'scram-sha-256';
    ```

1.  创建一个名为`repuser`的新用户：

    ```sql
    CREATE ROLE repuser WITH REPLICATION PASSWORD '<PASSWORD>' LOGIN;
    ```

<Highlight type="important">
[scram-sha-256](https://www.postgresql.org/docs/current/static/sasl-authentication.html#SASL-SCRAM-SHA-256) 加密级别是PostgreSQL中可用的最安全的基于密码的认证方式。它仅在PostgreSQL 10及更高版本中提供。
</Highlight>

</Procedure>

## 配置复制参数

有几个复制设置需要在`postgresql.conf`配置文件中添加或编辑。

<Procedure>

### 配置复制参数

1.  将`synchronous_commit`参数设置为`off`。
1.  将`max_wal_senders`参数设置为从副本或备份客户端的并发连接总数。作为最小值，这应该等于您打算拥有的副本数量。
1.  将`wal_level`参数设置为写入PostgreSQL预写式日志（WAL）的信息量。对于复制工作，WAL中需要有足够的数据来支持归档和复制。默认值通常合适。
1.  将`max_replication_slots`参数设置为主数据库可以支持的复制槽总数。
1.  将`listen_addresses`参数设置为主数据库的地址。不要将此参数留作本地回环地址，因为远程副本必须能够连接到主数据库以流式传输WAL。
1.  重启PostgreSQL以使更改生效。在创建复制槽之前必须执行此操作。

</Procedure>

最常见的流复制用例是具有一个或多个副本的异步复制。在这个示例中，WAL被流传输到副本，但主服务器不会等待确认WAL已在主服务器或副本上写入磁盘。这是性能最高的复制配置，但它确实存在在系统故障情况下丢失少量数据的风险。它也不保证副本与主数据库完全同步，这可能会导致在主数据库和副本上执行的读取查询之间存在不一致性。这种用例的示例配置如下：

```yaml
listen_addresses = '*'
wal_level = replica
max_wal_senders = 2
max_replication_slots = 2
synchronous_commit = off
```

如果您需要副本上更强的一致性，或者您的查询负载足够重，以至于在异步模式下主节点和副本节点之间出现显著的延迟，请考虑使用同步复制配置。有关不同复制模式的更多信息，请参见[复制模式部分][replication-modes]。

## 创建复制槽

配置了`postgresql.conf`并重启PostgreSQL后，您可以为每个副本创建[复制槽][postgres-rslots-docs]。复制槽确保主数据库在副本接收到WAL段之前不会删除WAL段。这在副本长时间停机的情况下很重要。主数据库需要验证WAL段已被副本消耗，以便它可以安全地删除数据。您可以使用[归档][postgres-archive-docs]来实现此目的，但复制槽为流复制提供了最强的保护。

<Procedure>

### 创建复制槽

1.  在`psql`槽中，创建第一个复制槽。槽的名称是任意的。在这个示例中，它被称为`replica_1_slot`：

    ```sql
    SELECT * FROM pg_create_physical_replication_slot('replica_1_slot', true);
    ```

1.  为每个所需的复制槽重复此操作。

</Procedure>

## 配置基于主机的认证参数

有几个复制设置需要添加或编辑到`pg_hba.conf`配置文件中。在这个示例中，设置限制了来自`REPLICATION_HOST_IP`作为PostgreSQL用户`repuser`的复制连接，使用有效密码。`REPLICATION_HOST_IP`可以从该机器启动流复制，无需额外的凭据。您可以更改`address`和`method`值以匹配您的安全和网络设置。

有关`pg_hba.conf`的更多信息，请参见[`pg_hba`文档][pg-hba-docs]。

<Procedure>

### 配置基于主机的认证参数

1.  打开`pg_hba.conf`配置文件，并添加或编辑此行：

    ```yaml
    TYPE  DATABASE    USER    ADDRESS METHOD            AUTH_METHOD
    host  replication repuser <REPLICATION_HOST_IP>/32  scram-sha-256
    ```

1.  重启PostgreSQL以使更改生效。

</Procedure>

## 在副本上创建基础备份

副本通过流传输主服务器的WAL日志并在PostgreSQL恢复模式下重放其事务来工作。为此，副本需要处于可以重放日志的状态。您可以通过从主实例的基础备份恢复副本来实现这一点。

<Procedure>

### 在副本上创建基础备份

1.  停止PostgreSQL服务。
1.  如果副本数据库已经包含数据，请在运行备份之前删除它，方法是移除PostgreSQL数据目录：

    ```bash
    rm -rf <DATA_DIRECTORY>/*
    ```

    如果您不知道数据目录的位置，请使用`show data_directory;`命令查找。
1.  从基础备份中恢复，使用主数据库的IP地址和复制用户名：

    ```bash
    pg_basebackup -h <PRIMARY_IP> \
    -D <DATA_DIRECTORY> \
    -U repuser -vP -W
    ```

    -W标志会提示您输入密码。如果您在自动化设置中使用此命令，您可能需要使用[pgpass文件][pgpass-file]。
1.  备份完成后，在数据目录中创建一个[standby.signal][postgres-recovery-docs]文件。当PostgreSQL在其数据目录中找到`standby.signal`文件时，它会以恢复模式启动并通过复制协议流式传输WAL：

    ```bash
    touch <DATA_DIRECTORY>/standby.signal
    ```

</Procedure>

## 配置复制和恢复设置

当您成功创建了基础备份和`standby.signal`文件后，您可以配置复制和恢复设置。

<Procedure>

### 配置复制和恢复设置

1.  在副本的`postgresql.conf`文件中，添加与主服务器通信的详细信息。如果您使用的是流复制，`primary_conninfo`中的`application_name`应与主服务器的`synchronous_standby_names`设置中使用的名称相同：

    ```yaml
    primary_conninfo = 'host=<PRIMARY_IP> port=5432 user=repuser
    password=<POSTGRES_USER_PASSWORD> application_name=r1'
    primary_slot_name = 'replica_1_slot'
    ```

1.  添加详细信息以镜像主数据库的配置。如果您使用的是异步复制，请使用这些设置：

    ```yaml
    hot_standby = on
    wal_level = replica
    max_wal_senders = 2
    max_replication_slots = 2
    synchronous_commit = off
    ```

    `hot_standby`参数必须设置为`on`，以允许在副本上执行只读查询。在PostgreSQL 10及更高版本中，此设置默认为`on`。
1.  重启PostgreSQL以使更改生效。

</Procedure>

## 验证副本是否正常工作

此时，您的副本应该已完全同步主数据库，并准备从其流式传输。您可以通过检查副本上的日志来验证其是否正常工作，日志应该如下所示：

```txt
LOG:  database system was shut down in recovery at 2018-03-09 18:36:23 UTC
LOG:  entering standby mode
LOG:  redo starts at 0/2000028
LOG:  consistent recovery state reached at 0/3000000
LOG:  database system is ready to accept read only connections
LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```

任何客户端都可以在副本上执行读取。您可以通过在主数据库上运行插入、更新或其他修改数据的操作，然后查询副本以确保它们已正确复制过来来验证这一点。

## 复制模式

在大多数情况下，异步流复制是足够的。然而，您可能需要主数据库和副本之间更强的一致性，特别是如果您的负载很重。在重负载下，副本可能会远远落后于主数据库，向从副本读取的客户端提供过时的数据。此外，在任何数据丢失都是致命的情况下，异步复制可能无法提供足够的持久性保证。PostgreSQL [`synchronous_commit`][postgres-synchronous-commit-docs]特性有几种选项，具有不同的一致性和性能权衡。

在`postgresql.conf`文件中，将`synchronous_commit`参数设置为：

*   `on`：这是默认值。服务器在主数据库和任何副本上将WAL事务写入磁盘之前不会返回`success`。
*   `off`：当WAL事务已发送到操作系统写入主数据库磁盘上的WAL时，服务器返回`success`，但不等待操作系统实际写入。如果服务器在一些数据尚未写入时崩溃，这可能会导致少量数据丢失，但不会导致数据损坏。关闭`synchronous_commit`是众所周知的PostgreSQL优化，适用于能够承受系统崩溃时一些数据丢失的工作负载。
*   `local`：仅在主服务器上强制`on`行为。
*   `remote_write`：当WAL记录已发送到操作系统以写入副本上的WAL时，数据库向客户端返回`success`，但在确认记录实际已持久保存到磁盘之前。这类似于异步提交，除了它也等待副本以及主数据库。实际上，等待副本的额外等待时间显著减少了复制延迟。
*   `remote_apply`：需要确认WAL记录已写入所有副本上的WAL并应用于数据库。这提供了`synchronous_commit`选项中最强的一致性。在此模式下，副本始终反映主数据库的最新状态，复制延迟几乎不存在。

<Highlight type="important">
如果`synchronous_standby_names`为空，则设置`on`、`remote_apply`、`remote_write`和`local`都提供相同的同步级别，事务提交等待本地磁盘刷新。
</Highlight>

这个矩阵展示了每种模式提供的一致性水平：

| 模式 | WAL 发送到操作系统（主服务器） | WAL 在操作系统中持久化（主服务器） | WAL 发送到操作系统（主服务器和副本服务器） | WAL 在操作系统中持久化（主服务器和副本服务器） | 事务应用（主服务器和副本服务器） |
|-|-|-|-|-|-|
| Off | ✅ | ❌ | ❌ | ❌ | ❌ |
| Local | ✅ | ✅ | ❌ | ❌ | ❌ |
| Remote Write | ✅ | ✅ | ✅ | ❌ | ❌ |
| On | ✅ | ✅ | ✅ | ✅ | ❌ |
| Remote Apply | ✅ | ✅ | ✅ | ✅ | ✅ |

`synchronous_standby_names` 设置是 `synchronous_commit` 的补充设置。它列出了主数据库支持的所有副本服务器的名称，并配置了主数据库等待它们的方式。`synchronous_standby_names` 设置支持以下格式：

*   `FIRST num_sync (replica_name_1, replica_name_2)`: 这在返回 `success` 之前等待来自前 `num_sync` 个副本服务器的确认。`replica_names` 列表决定了副本服务器的相对优先级。副本服务器的名称由副本服务器上的 `application_name` 设置确定。
*   `ANY num_sync (replica_name_1, replica_name_2)`: 这在提供的列表中等待 `num_sync` 个副本服务器的确认，无论它们在列表中的优先级或位置如何。这作为一个法定人数功能工作。

同步复制模式强制主服务器等待所有必需的副本服务器写入WAL，或应用数据库事务，具体取决于 `synchronous_commit` 级别。如果所需的副本服务器崩溃，这可能会导致主服务器无限期挂起。当副本服务器重新连接时，它会回放任何需要赶上的WAL。只有这样，主服务器才能恢复写入。为了缓解这种情况，在 `synchronous_standby_names` 设置下多准备一些节点，并在 `FIRST` 或 `ANY` 条款中列出它们。这允许主服务器只要法定人数的副本服务器写入了最新的WAL事务就能向前推进。服务中断的副本服务器能够重新连接并异步回放错过的WAL事务。

## 复制诊断

PostgreSQL [pg_stat_replication][postgres-pg-stat-replication-docs] 视图提供了关于每个副本服务器的信息。这个视图特别适用于计算复制延迟，它衡量了副本服务器当前状态比主服务器落后多远。`replay_lag` 字段给出了主服务器上最近的WAL事务和副本服务器上最后报告的数据库提交之间的秒数。结合 `write_lag` 和 `flush_lag`，这提供了副本服务器落后多远的洞察。`*_lsn` 字段也提供了有用的信息。它们允许你比较主服务器和副本服务器之间的WAL位置。`state` 字段有助于确定每个副本服务器当前正在做什么；可用的模式有 `startup`、`catchup`、`streaming`、`backup` 和 `stopping`。

要查看数据，在主数据库上运行此命令：

```sql
SELECT * FROM pg_stat_replication;
```

输出如下所示：

```sql
-[ RECORD 1 ]----+------------------------------
pid              | 52343
usesysid         | 16384
usename          | repuser
application_name | r2
client_addr      | 10.0.13.6
client_hostname  | 
client_port      | 59610
backend_start    | 2018-02-07 19:07:15.261213+00
backend_xmin     | 
state           | streaming
sent_lsn        | 16B/43DB36A8
write_lsn       | 16B/43DB36A8
flush_lsn       | 16B/43DB36A8
replay_lsn      | 16B/43107C28
write_lag       | 00:00:00.009966
flush_lag       | 00:00:00.03208
replay_lag      | 00:00:00.43537
sync_priority   | 2
sync_state      | sync
-[ RECORD 2 ]----+------------------------------
pid             | 54498
usesysid         | 16384
usename         | repuser
application_name| r1
client_addr     | 10.0.13.5
client_hostname | 
client_port     | 43402
backend_start   | 2018-02-07 19:45:41.410929+00
backend_xmin    | 
state          | streaming
sent_lsn        | 16B/43DB36A8
write_lsn       | 16B/43DB36A8
flush_lsn       | 16B/43DB36A8
replay_lsn      | 16B/42C3B9C8
write_lag       | 00:00:00.019736
flush_lag       | 00:00:00.044073
replay_lag      | 00:00:00.644004
sync_priority   | 1
sync_state      | sync
```

## 故障转移

PostgreSQL提供了一些故障转移功能，其中副本服务器在发生故障时被提升为主服务器。这是通过 [pg_ctl][pgctl-docs] 命令或 `trigger_file` 提供的。然而，PostgreSQL不提供自动故障转移的支持。有关更多信息，请参见 [PostgreSQL故障转移文档][failover-docs]。如果您需要具有自动故障转移功能的可配置高可用性解决方案，请查看 [Patroni][patroni-github]。

[configure-params]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#configure-replication-parameters
[configure-pghba]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#configure-host-based-authentication-parameters
[configure-primary-db]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#configure-the-primary-database
[configure-replication]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#configure-replication-and-recovery-settings
[create-base-backup]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#create-a-base-backup-on-the-replica
[create-replication-slots]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#create-replication-slots
[docker-postgres-scripts]: https://hub.docker.com/_/postgres/  
[failover-docs]: https://www.postgresql.org/docs/current/static/warm-standby-failover.html  
[patroni-github]: https://github.com/zalando/patroni  
[pg-hba-docs]: https://www.postgresql.org/docs/current/static/auth-pg-hba-conf.html  
[pgctl-docs]: https://www.postgresql.org/docs/current/static/app-pg-ctl.html  
[pgpass-file]: https://www.postgresql.org/docs/current/libpq-pgpass.html  
[postgres-archive-docs]: https://www.postgresql.org/docs/current/static/continuous-archiving.html  
[postgres-pg-stat-replication-docs]: https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW  
[postgres-recovery-docs]: https://www.postgresql.org/docs/current/runtime-config-wal.html#RUNTIME-CONFIG-WAL-ARCHIVE-RECOVERY  
[postgres-rslots-docs]: https://www.postgresql.org/docs/current/static/warm-standby.html#STREAMING-REPLICATION-SLOTS  
[postgres-synchronous-commit-docs]: https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-SYNCHRONOUS-COMMIT  
[replication-modes]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#replication-modes
[timescale-streamrep-helm]: https://github.com/timescale/helm-charts/tree/main/charts/timescaledb-single  
[verify-replica]: /self-hosted/:currentVersion:/replication-and-ha/configure-replication#verify-that-the-replica-is-working


