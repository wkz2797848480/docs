---
标题: 使用 Docker 进行配置
摘要: 配置在 Docker 容器中运行的 TimescaleDB 实例
产品: [自托管]
关键词: [配置，设置，Docker]
---

# 在Docker中配置

如果您在[Docker容器][docker]中运行TimescaleDB，有两种不同的方式可以修改您的PostgreSQL配置。您可以直接在Docker容器内编辑PostgreSQL配置文件，或者您也可以在命令行中设置参数。

## 在Docker内编辑PostgreSQL配置文件

您可以启动Docker容器，然后使用文本编辑器直接编辑PostgreSQL配置文件。配置文件需要每行一个参数。空行会被忽略，您可以在行首使用`#`符号来表示注释。

<Procedure>

### 在Docker内编辑PostgreSQL配置文件

1.  启动您的Docker实例：

    ```bash
    docker start timescaledb
    ```

1.  打开shell：

    ```bash
    docker exec -i -t timescaledb /bin/bash
    ```

1.  在`Vi`编辑器或您偏好的文本编辑器中打开配置文件。

    ```bash
    vi /var/lib/postgresql/data/postgresql.conf
    ```

1.  重启容器以重新加载配置：

    ```bash
    docker restart timescaledb
    ```

</Procedure>

## 在命令行中设置参数

如果您不想打开配置文件进行更改，您也可以在Docker容器内的命令行中直接使用`-c`选项设置参数。例如：

```bash
docker run -i -t timescale/timescaledb:latest-pg10 postgres -c max_wal_size=2GB
```

[docker]: /self-hosted/latest/install/installation-docker/

