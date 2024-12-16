---
标题: 在 Docker 上安装 TimescaleDB
摘要: 从预先构建的 Docker 容器中安装自托管的 TimescaleDB
产品: [自托管]
关键词: [安装，自托管，Docker]
---

import WhereTo from "versionContent/_partials/_where-to-next.mdx";
import Skip from "versionContent/_partials/_selfhosted_cta.mdx";
import SelfHostedDocker from "versionContent/_partials/_install-self-hosted-docker-based.mdx";
import AddTimescaleDBToDB from "versionContent/_partials/_add-timescaledb-to-a-database.mdx";

# 从 Docker 容器安装 TimescaleDB

TimescaleDB 是一个 [PostgreSQL 扩展](https://www.postgresql.org/docs/current/external-extensions.html)，用于时间序列和高吞吐量的数据查询和摄入。你可以从预构建的 Docker 容器在任何本地系统上安装 TimescaleDB 实例。

<跳过 />

本节展示了如何[在 PostgreSQL 上安装和配置 TimescaleDB](#在-postgresql 上安装和配置-timescaledb)。

### 前提条件

要运行和连接到 Docker 上的 PostgreSQL 安装，你需要安装：

- [Docker][docker-install]
- [psql][install-psql]

## 在 PostgreSQL 上安装和配置 TimescaleDB

本节展示了如何使用 TimeScale 提供的容器在[支持的平台](#支持的平台)上使用容器安装最新版本的 PostgreSQL 和 TimescaleDB。

<SelfHostedDocker />

就是这样！你已经在 PostgreSQL 的自托管实例上运行了 TimescaleDB。

## 更多 Docker 选项

如果你想直接从容器运行映像，你可以使用这个命令：

```bash
docker run -d --name timescaledb -p 5432:5432 -e POSTGRES_PASSWORD=password timescale/timescaledb-ha:pg17
```

`-p` 标志将容器端口绑定到主机端口。这意味着任何可以访问主机端口的东西也可以访问你的 TimescaleDB 容器，所以重要的是你设置一个 PostgreSQL 密码使用 `POSTGRES_PASSWORD` 环境变量。如果没有那个变量，Docker 容器会禁用所有数据库用户的身份验证检查。

如果你想从主机访问容器但避免将其暴露给外界，你可以绑定到 `127.0.0.1` 而不是公共接口，使用这个命令：

```bash
docker run -d --name timescaledb -p 127.0.0.1:5432:5432 \
-e POSTGRES_PASSWORD=password timescale/timescaledb-ha:pg17
```

如果你不想在本地安装 `psql` 和其他 PostgreSQL 客户端工具，或者如果你使用的是 Microsoft Windows 主机系统，你可以使用容器中捆绑的 `psql` 版本连接，用这个命令：

```bash
docker exec -it timescaledb psql -U postgres
```

可以使用 `docker stop` 停止现有容器，并使用 `docker start` 重新启动它们，同时保留它们的卷和数据。当你使用 `docker run` 命令创建一个新的容器时，默认情况下你也会创建一个新的数据卷。当你使用 `docker rm` 删除 Docker 容器时，数据卷会保留在磁盘上，直到你显式地删除它。你可以使用 `docker volume ls` 命令列出现有的 Docker 卷。如果你想将你的 Docker 容器的数据存储在主机目录中，或者你想在现有的数据目录上运行 Docker 映像，你可以使用 `-v` 标志指定目录来挂载一个数据卷。

<Highlight type="warning">
两种容器类型在不同的地方存储 PostgreSQL 数据目录，确保你选择正确的一个来挂载：

<!-- vale Vale.Terms = NO -->
|容器|PGDATA位置|
|-|-|
`timescaledb-ha`|`/home/postgres/pgdata/data`
`timescaledb`| `/var/lib/postgresql/data`
<!-- vale Vale.Terms = YES -->
</Highlight>

```bash
docker run -d --name timescaledb -p 5432:5432 \
-v /your/data/dir:/home/postgres/pgdata/data \
-e POSTGRES_PASSWORD=password timescale/timescaledb-ha:pg17
```

当你使用 Docker 容器安装 TimescaleDB 时，PostgreSQL 设置是从容器继承的。在大多数情况下，你不需要调整它们。然而，如果你需要更改设置，你可以添加 `-c setting=value` 到你的 Docker `run` 命令。更多信息，请参阅[Docker 文档][docker-postgres]。

这些说明中提供的链接是 PostgreSQL 16 上 TimescaleDB 最新版本的链接。要查找你可以使用的其他 Docker 标签，请参阅[Dockerhub 仓库][dockerhub]。

### 在 Docker 中查看日志

如果你在 Docker 容器中安装了 TimescaleDB，你可以使用 Docker 查看日志，而不是查看 `/var/lib/logs` 或 `/var/logs`。更多信息，请参阅[Docker 日志文档][docker-logs]。

## 接下来做什么

<WhereTo />

[alpine]: https://alpinelinux.org/  
[config]: /self-hosted/:currentVersion:/configuration/
[docker-install]: https://docs.docker.com/get-docker/  
[docker-postgres]: https://hub.docker.com/_/postgres  
[dockerhub]: https://hub.docker.com/r/timescale/timescaledb/tags?page=1&ordering=last_updated  
[install-psql]: https://www.timescale.com/blog/how-to-install-psql-on-mac-ubuntu-debian-windows/  
[ubuntu]: https://ubuntu.com  
[docker-logs]: https://docs.docker.com/config/containers/logging/  
[install-from-source]: /self-hosted/:currentVersion:/install/installation-source/
