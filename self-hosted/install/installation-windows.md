---
标题: 在 Windows 系统上安装 TimescaleDB
摘要: 在 Windows 系统上安装自托管的 TimescaleDB
产品: [自托管]
关键词: [安装，自托管，Windows]
---

import Windows from "versionContent/_partials/_psql-installation-windows.mdx";
import WhereTo from "versionContent/_partials/_where-to-next.mdx";
import Skip from "versionContent/_partials/_selfhosted_cta.mdx";
import SelfHostedWindowsBased from "versionContent/_partials/_install-self-hosted-windows-based.mdx";
import AddTimescaleDBToDB from "versionContent/_partials/_add-timescaledb-to-a-database.mdx";

# 在 Windows 上安装 TimescaleDB

TimescaleDB 是一个 [PostgreSQL 扩展](https://www.postgresql.org/docs/current/external-extensions.html)，用于时间序列和高吞吐量的数据查询和摄入。

<跳过 />

本节展示了如何：

* [在 PostgreSQL 上安装和配置 TimescaleDB][install-timescaledb]：设置自托管的 PostgreSQL 实例以高效运行 TimescaleDB。
* [在数据库中添加 TimescaleDB 扩展][add-timescledb-extension]：在数据库上启用 TimescaleDB 特性和性能改进。

<Highlight type="warning">

如果您之前在没有包管理器的情况下安装了 PostgreSQL，您可能会在遵循这些安装说明时遇到错误。最佳实践是在开始之前完全移除任何现有的 PostgreSQL 安装。

要保留您当前的 PostgreSQL 安装，请[从源代码安装][install-from-source]。

</Highlight>

### 前提条件

要在 Windows 设备上安装 TimescaleDB，您需要：

* OpenSSL v3.x
* [Visual C++ Redistributable for Visual Studio 2015][ms-download]

## 在 PostgreSQL 上安装和配置 TimescaleDB

本节展示了如何使用 TimeScale 提供的包在[支持的平台][supported-platforms]上安装最新版本的 PostgreSQL 和 TimescaleDB。

<SelfHostedWindowsBased />

## 在数据库中添加 TimescaleDB 扩展

为了提高性能，您需要在自托管的 PostgreSQL 实例上的每个数据库上启用 TimescaleDB。
本节展示了如何使用命令行中的 `psql` 为 PostgreSQL 中的新数据库启用 TimescaleDB。

<AddTimescaleDBToDB />

就是这样！您已经在 PostgreSQL 的自托管实例上的数据库上运行了 TimescaleDB。

## 接下来做什么

<WhereTo />

## 支持的平台

* TimescaleDB 支持的 PostgreSQL 版本 13、14、15 和 16 的最新版本分别为：

    *   <Tag type="download">
        [PostgreSQL 17: Timescale 发布](https://github.com/timescale/timescaledb/releases/latest/download/timescaledb-postgresql-17-windows-amd64.zip) 
        </Tag>
    *   <Tag type="download">
        [PostgreSQL 16: Timescale 发布](https://github.com/timescale/timescaledb/releases/latest/download/timescaledb-postgresql-16-windows-amd64.zip) 
        </Tag>
    *   <Tag type="download">
        [PostgreSQL 15: Timescale 发布](https://github.com/timescale/timescaledb/releases/latest/download/timescaledb-postgresql-15-windows-amd64.zip) 
        </Tag>
    *   <Tag type="download">
        [PostgreSQL 14: Timescale 发布](https://github.com/timescale/timescaledb/releases/latest/download/timescaledb-postgresql-14-windows-amd64.zip) 
        </Tag>
    *   <Tag type="download">
        [PostgreSQL 13: Timescale 发布](https://github.com/timescale/timescaledb/releases/latest/download/timescaledb-postgresql-13-windows-amd64.zip) 
        </Tag>

* TimescaleDB 支持以下平台：

  *   Microsoft Windows 10
  *   Microsoft Windows 11
  *   Microsoft Windows Server 2019

有关发布信息，请查看 [GitHub 发布页面][gh-releases] 和 [发布说明][release-notes]。

[config]: /self-hosted/:currentVersion:/configuration/
[gh-releases]: https://github.com/timescale/timescaledb/releases 
[ms-download]: https://www.microsoft.com/en-us/download/details.aspx?id=48145 
[pg-download]: https://www.postgresql.org/download/windows/ 
[release-notes]: https://github.com/timescale/timescaledb/releases 
[windows-releases]: #windows-releases
[install-from-source]: /self-hosted/:currentVersion:/install/installation-source/
[install-timescaledb]: /self-hosted/:currentVersion:/install/installation-windows/#install-and-configure-timescaledb-on-postgresql
[add-timescledb-extension]: /self-hosted/:currentVersion:/install/installation-windows/#add-the-timescaledb-extension-to-your-database
[supported-platforms]: /self-hosted/:currentVersion:/install/installation-windows/#supported-platforms
