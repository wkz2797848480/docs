---
标题: 安装和更新 TimescaleDB 工具包
摘要: 如何安装 TimescaleDB 工具包以使用更多超函数和函数管道
产品: [托管服务（mst），自托管]
关键词: [工具包，安装，超函数，函数管道]
---

# 安装和更新TimescaleDB Toolkit

Timescale默认包含了一些超函数。若需要更多的超函数，您需要安装TimescaleDB Toolkit PostgreSQL扩展。

如果您使用的是[Timescale][cloud]，Toolkit已经预安装好了。

## 在TimescaleDB管理服务上安装和更新Toolkit

在[TimescaleDB管理服务][mst]上，对您希望使用Toolkit的每个数据库运行以下命令：

```sql
CREATE EXTENSION timescaledb_toolkit;
```

使用此命令更新已安装的Toolkit版本：

```sql
ALTER EXTENSION timescaledb_toolkit UPDATE;
```

## 在自托管的TimescaleDB上安装Toolkit

如果您托管自己的TimescaleDB数据库，可以通过以下方式安装Toolkit：

*   使用TimescaleDB高可用性Docker镜像
*   在提供预构建二进制文件的平台使用包管理器如`yum`、`apt`或`brew`
*   从源代码构建

### 安装Docker镜像

推荐使用[TimescaleDB Docker镜像](https://github.com/timescale/timescaledb-docker-ha)来安装Toolkit。
要获取Toolkit，请使用高可用性镜像`timescaledb-ha`：

```bash
docker pull timescale/timescaledb-ha:pg17
```

有关使用Docker运行TimescaleDB的更多信息，请参见[预构建容器][docker-install]部分。

### 在CentOS 7和其他基于Red Hat的系统上安装Toolkit

这些说明使用`yum`包管理器。它们已在CentOS 7上测试过，可能也适用于其他基于Red Hat的系统，如Red Hat Enterprise Linux和Fedora。

<Procedure>

#### 在CentOS 7上安装Toolkit

1.  确保您已安装TimescaleDB并在您的`yum` `repo.d`目录中创建了TimescaleDB仓库。更多信息，请参见[基于Red Hat系统的安装说明][red-hat-install]。
1.  更新本地仓库列表：

    ```bash
    yum update
    ```

1.  安装TimescaleDB Toolkit：

    ```bash
    yum install timescaledb-toolkit-postgresql-16
    ```

1.  连接到您希望使用Toolkit的数据库。
1.  在数据库中创建Toolkit扩展：

    ```sql
    CREATE EXTENSION timescaledb_toolkit;
    ```

</Procedure>

### 在Ubuntu和其他基于Debian的系统上安装Toolkit

这些说明使用`apt`包管理器。它们已在Ubuntu 20.04上测试过，可能也适用于其他基于Debian的系统。

<Procedure>

#### 在Ubuntu 20.04上安装Toolkit

1.  确保您已安装TimescaleDB并添加了TimescaleDB仓库和GPG密钥。更多信息，请参见[基于Debian系统的安装说明][debian-install]。
1.  更新本地仓库列表：

    ```bash
    apt update
    ```

1.  安装TimescaleDB Toolkit：

    ```bash
    apt install timescaledb-toolkit-postgresql-16
    ```

1.  连接到您希望使用Toolkit的数据库。
1.  在数据库中创建Toolkit扩展：

    ```sql
    CREATE EXTENSION timescaledb_toolkit;
    ```

</Procedure>

### 在macOS上安装Toolkit

这些说明使用`brew`包管理器。有关安装或使用Homebrew的更多信息，请参见[`brew`首页][brew-install]。

<Procedure>

#### 在macOS上安装Toolkit

1.  Tap Timescale公式仓库，其中也包含了TimescaleDB和`timescaledb-tune`的公式。

    ```bash
    brew tap timescale/tap
    ```

1.  更新您的本地brew安装：

    ```bash
    brew update
    ```

1.  安装TimescaleDB Toolkit：

    ```bash
    brew install timescaledb-toolkit
    ```

1.  连接到您希望使用Toolkit的数据库。
1.  在数据库中创建Toolkit扩展：

    ```sql
    CREATE EXTENSION timescaledb_toolkit;
    ```

</Procedure>

### 在Windows上安装Toolkit

TimescaleDB Toolkit目前不支持Windows。作为变通方法，您可以在Docker容器中运行PostgreSQL。

## 在自托管的TimescaleDB上更新Toolkit

通过安装最新版本并运行`ALTER EXTENSION`来更新Toolkit。

<Procedure>

### 在自托管的TimescaleDB上更新Toolkit

1.  更新您的本地仓库列表：

    <Terminal>

    <tab label='CentOS 7'>

    ```bash
    yum update
    ```

    </tab>

    <tab label='Debian'>

    ```bash
    apt update
    ```

    </tab>

    <tab label='macOS'>

    ```bash
    brew update
    ```

    </tab>

    </Terminal>

1.  安装最新版本的TimescaleDB Toolkit：

    <Terminal>

    <tab label='CentOS 7'>

    ```bash
    yum install timescaledb-toolkit-postgresql-16
    ```

    </tab>

    <tab label='Debian'>

    ```bash
    apt install timescaledb-toolkit-postgresql-16
    ```

    </tab>

    <tab label='macOS'>

    ```bash
    brew upgrade timescaledb-toolkit
    ```

    </tab>

    </Terminal>

1.  连接到您希望使用新版本Toolkit的数据库。
1.  在数据库中更新Toolkit扩展：

    ```sql
    ALTER EXTENSION timescaledb_toolkit UPDATE;
    ```

<Highlight type="note">
对于某些Toolkit版本，您可能需要断开并重新连接活动会话。
</Highlight>

</Procedure>

### 从源代码构建Toolkit

您可以从源代码构建Toolkit。更多信息，请参见[Toolkit开发者文档][toolkit-gh-docs]。

[brew-install]: https://brew.sh 
[cloud]: /use-timescale/:currentVersion:/services/
[debian-install]: /self-hosted/latest/install/installation-linux/
[docker-install]: /self-hosted/latest/install/installation-docker/
[mst]: /mst/:currentVersion:/
[red-hat-install]: /self-hosted/latest/install/installation-linux/
[toolkit-gh-docs]: https://github.com/timescale/timescaledb-toolkit#-installing-from-source
