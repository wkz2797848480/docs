---
标题: 手动 PostgreSQL 配置与调优
摘要: 如何手动配置您的 PostgreSQL 实例
产品: [自托管]
关键词: [配置，设置]
标签: [调优]
---

# 手动PostgreSQL配置和调优

如果您更倾向于自己调整设置，或者对于`timescaledb-tune`未覆盖的设置，您可以使用PostgreSQL配置文件手动配置您的安装。

对于您可能想要调整的一些常见配置设置，请参见[关于配置][about-configuration]页面。

有关PostgreSQL配置页面的更多信息，请参见[PostgreSQL文档][pg-config]。

## 编辑PostgreSQL配置文件

PostgreSQL配置文件的位置取决于您的操作系统和安装方式。您可以通过以`postgres`用户身份从psql提示符查询数据库来找到位置：

```sql
SHOW config_file;
```

配置文件需要每行一个参数。空行会被忽略，您可以在行首使用`#`符号来表示注释。

当您对配置文件进行了更改后，新的配置不会立即应用。每当服务器接收到`SIGHUP`信号时，配置文件会被重新加载，或者您可以手动使用`pg_ctl`命令重新加载文件。

## 在命令行中设置参数

如果您不想打开配置文件进行更改，您也可以直接从命令行使用`postgres`命令设置参数。例如：

```sql
postgres -c log_connections=yes -c log_destination='syslog'
```

[about-configuration]: /self-hosted/:currentVersion:/configuration/about-configuration
[pg-config]: https://www.postgresql.org/docs/current/config-setting.html

