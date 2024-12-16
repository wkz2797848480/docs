---
标题: 使用 Docker 和 WAL-E 进行持续物理备份
摘要: 备份你的 TimescaleDB 的 Docker 实例
产品: [自托管]
关键词: [备份，Docker]
标签: [恢复，复原，物理备份]
---

import 弃用警告 from "versionContent/_partials/_deprecated.mdx";
import 考虑使用云服务 from "versionContent/_partials/_consider-cloud.mdx";


# 使用Docker和WAL-E进行持续物理备份

当您在容器化环境中运行TimescaleDB时，可以使用[持续归档][pg archiving]功能，配合[WAL-E][wale official]容器。这些容器有时被称为边车（sidecars），因为它们与主容器一起运行。[WAL-E边车镜像][wale image]可以与TimescaleDB以及常规的PostgreSQL一起工作。在本节中，您可以将归档设置到本地文件系统，使用名为`timescaledb`的主TimescaleDB容器和名为`wale`的WAL-E边车。当您准备在生产部署中实施此方案时，可以按照这里的说明适应云服务提供商（如AWS S3）的归档，并在Kubernetes等编排框架中运行。

<ConsiderCloud />

## 在Docker中运行TimescaleDB容器

要使TimescaleDB使用WAL-E边车进行归档，两个容器需要共享网络。为此，您需要创建一个Docker网络，然后使用新创建的网络启动已开启归档功能的TimescaleDB。启动TimescaleDB时，您需要明确设置预写日志（write-ahead log，`POSTGRES_INITDB_WALDIR`）和数据目录（`PGDATA`）的位置，以便与WAL-E边车共享。两者都必须位于Docker卷中，默认情况下，会为`/var/lib/postgresql/data`创建一个卷。启动TimescaleDB后，您可以登录并创建表和数据。

<Deprecation />

<Procedure>

### 在Docker中运行TimescaleDB容器

1.  创建docker容器：

    ```bash
    docker network create timescaledb-net
    ```

1.  启动TimescaleDB，开启归档功能：

    ```bash
    docker run \
      --name timescaledb \
      --network timescaledb-net \
      -e POSTGRES_PASSWORD=insecure \
      -e POSTGRES_INITDB_WALDIR=/var/lib/postgresql/data/pg_wal \
      -e PGDATA=/var/lib/postgresql/data/pg_data \
      timescale/timescaledb:latest-pg10 postgres \
      -cwal_level=archive \
      -carchive_mode=on \
      -carchive_command="/usr/bin/wget wale/wal-push/%f -O -" \
      -carchive_timeout=600 \
      -ccheckpoint_timeout=700 \
      -cmax_wal_senders=1
    ```

1.  在Docker中运行TimescaleDB：

    ```bash
    docker exec -it timescaledb psql -U postgres
    ```

</Procedure>

## 使用WAL-E边车执行备份

[WAL-E Docker镜像][wale image]运行一个Web端点，接受通过HTTP API发送的WAL-E命令。这允许PostgreSQL通过内部网络与WAL-E边车通信以触发归档。您也可以直接使用容器调用WAL-E。Docker镜像接受标准的WAL-E环境变量来配置归档后端，因此您可以从AWS S3等服务发出命令。有关配置信息，请参见官方[WAL-E文档][wale official]。

要使WAL-E Docker镜像执行归档，它需要使用与TimescaleDB容器相同的网络和数据卷。它还需要知道预写日志和数据目录的位置。当您启动WAL-E时，可以将所有这些信息传递给它。在此示例中，WAL-E镜像在`timescaledb-net`内部网络上的端口80上侦听命令，并将备份写入Docker主机上的`~/backups`。

<Procedure>

### 使用WAL-E边车执行备份

1.  使用有关容器的所需信息启动WAL-E容器。
    在此示例中，容器名为`timescaledb-wale`：

    ```bash
    docker run \
      --name wale \
      --network timescaledb-net \
      --volumes-from timescaledb \
      -v ~/backups:/backups \
      -e WALE_LOG_DESTINATION=stderr \
      -e PGWAL=/var/lib/postgresql/data/pg_wal \
      -e PGDATA=/var/lib/postgresql/data/pg_data \
      -e PGHOST=timescaledb \
      -e PGPASSWORD=insecure \
      -e PGUSER=postgres \
      -e WALE_FILE_PREFIX=file://localhost/backups \
      timescale/timescaledb-wale:latest
    ```

1.  启动备份：

    ```bash
    docker exec wale wal-e backup-push /var/lib/postgresql/data/pg_data
    ```

    或者，您可以使用边车的HTTP端点启动备份。
    这需要将边车的端口80暴露在Docker主机上，并将其映射到一个开放的端口。
    在此示例中，它被映射到端口8080：

    ```bash
    curl http://localhost:8080/backup-push
    ```

</Procedure>

您应该定期每日进行基础备份，以最小化WAL-E重播的数量，并加快恢复速度。
要制作新的基底备份，请如上所示手动或按计划重新触发基底备份。
如果您在Kubernetes上运行TimescaleDB，有内置的支持用于调度cron作业，这些作业可以调用WAL-E容器的HTTP API来执行基底备份。

## 恢复

要从备份存档中恢复数据库实例，请创建一个新的TimescaleDB容器，并从基底备份中恢复数据库和配置文件。
然后您可以重新启动边车和数据库。

<Procedure>

### 从备份恢复数据库文件

1.  创建docker容器：

    ```bash
    docker create \
      --name timescaledb-recovered \
      --network timescaledb-net \
      -e POSTGRES_PASSWORD=insecure \
      -e POSTGRES_INITDB_WALDIR=/var/lib/postgresql/data/pg_wal \
      -e PGDATA=/var/lib/postgresql/data/pg_data \
      timescale/timescaledb:latest-pg10 postgres
    ```

1.  从基底备份恢复数据库文件：

    ```bash
    docker run -it --rm \
      -v ~/backups:/backups \
      --volumes-from timescaledb-recovered \
      -e WALE_LOG_DESTINATION=stderr \
      -e WALE_FILE_PREFIX=file://localhost/backups \
      timescale/timescaledb-wale:latest \wal-e \
      backup-fetch /var/lib/postgresql/data/pg_data LATEST
    ```

1.  重新创建配置文件。这些文件是从原始数据库实例中备份的：

    ```bash
    docker run -it --rm  \
      --volumes-from timescaledb-recovered \
      timescale/timescaledb:latest-pg10 \
      cp /usr/local/share/postgresql/pg_ident.conf.sample /var/lib/postgresql/data/pg_data/pg_ident.conf

    docker run -it --rm  \
      --volumes-from timescaledb-recovered \
      timescale/timescaledb:latest-pg10 \

    cp /usr/local/share/postgresql/postgresql.conf.sample /var/lib/postgresql/data/pg_data/postgresql.conf

    docker run -it --rm  \
      --volumes-from timescaledb-recovered \
      timescale/timescaledb:latest-pg10 \

    sh -c 'echo "local all postgres trust" > /var/lib/postgresql/data/pg_data/pg_hba.conf'
    ```

1.  创建一个`recovery.conf`文件，告诉PostgreSQL如何恢复：

    ```bash
    docker run -it --rm  \
      --volumes-from timescaledb-recovered \
      timescale/timescaledb:latest-pg10 \

    sh -c 'echo "restore_command='\''/usr/bin/wget wale/wal-fetch/%f -O -'\''" > /var/lib/postgresql/data/pg_data/recovery.conf'
    ```

</Procedure>

当您恢复了数据和配置文件，并创建了恢复配置文件后，可以重新启动边车。
您可能需要先删除旧的边车。
当您重新启动边车时，它会重播可能不在基底备份中的最后WAL段。
然后您可以重新启动数据库，并检查恢复是否成功。

<Procedure>

### 重新启动已恢复的数据库

1. 重新启动WAL-E辅助容器：

    ```bash
    docker run \
      --name wale \
      --network timescaledb-net \
      -v ~/backups:/backups \
      --volumes-from timescaledb-recovered \
      -e WALE_LOG_DESTINATION=stderr \
      -e PGWAL=/var/lib/postgresql/data/pg_wal \
      -e PGDATA=/var/lib/postgresql/data/pg_data \
      -e PGHOST=timescaledb \
      -e PGPASSWORD=insecure \
      -e PGUSER=postgres \
      -e WALE_FILE_PREFIX=file://localhost/backups \
      timescale/timescaledb-wale:latest
    ```

2. 重新启动TimescaleDB docker容器：

    ```bash
    docker start timescaledb-recovered
    ```

3. 验证数据库是否成功启动和恢复：

    ```bash
    docker logs timescaledb-recovered
    ```

    如果在日志中看到一些归档恢复错误，请不要担心。这是因为直到在归档中找不到更多文件之前，恢复操作并未完全结束。有关更多信息，请参见PostgreSQL文档中的[持续归档][pg archiving]。

</Procedure>

[pg archiving]: https://www.postgresql.org/docs/current/continuous-archiving.html#BACKUP-PITR-RECOVERY 
[wale image]: https://hub.docker.com/r/timescale/timescaledb-wale 
[wale official]: https://github.com/wal-e/wal-e
