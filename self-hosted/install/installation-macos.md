---
标题: 在 macOS 系统上安装 TimescaleDB
摘要: 在 macOS 系统上安装自托管的 TimescaleDB
产品: [自托管]
关键词: [安装，自托管，macOS]
---

import WhereTo from "versionContent/_partials/_where-to-next.mdx";
import Skip from "versionContent/_partials/_selfhosted_cta.mdx";
import SelfHostedHomebrew from "versionContent/_partials/_install-self-hosted-homebrew-based.mdx";
import SelfHostedMacports from "versionContent/_partials/_install-self-hosted-macports-based.mdx";
import AddTimescaleDBToDB from "versionContent/_partials/_add-timescaledb-to-a-database.mdx";

# 在macOS上自行托管TimescaleDB安装

TimescaleDB是一个[PostgreSQL扩展](https://www.postgresql.org/docs/current/external-extensions.html)，用于时间序列和高吞吐量数据查询的工作负载。你可以在macOS设备上托管TimescaleDB。

<跳过 />

本节向您展示如何：

* [在PostgreSQL上安装和配置TimescaleDB](#在postgresql上安装和配置timescaledb) - 设置自托管的PostgreSQL实例以高效运行TimescaleDB。
* [在数据库中添加TimescaleDB扩展](#在数据库中添加timescaledb扩展) - 在数据库上启用TimescaleDB功能和性能改进。

### 前提条件

要在您的MacOS设备上安装TimescaleDB，您需要：

* [PostgreSQL][install-postgresql]：为了获得最新功能，请安装PostgreSQL v16

<Highlight type="warning">
如果您已经使用除Homebrew或MacPorts之外的方法安装了PostgreSQL，您可能会在遵循这些安装说明时遇到错误。最佳实践是在开始之前完全移除任何现有的PostgreSQL安装。

如果您想保留当前的PostgreSQL安装，请[从源代码安装][install-from-source]。
</Highlight>

## 在PostgreSQL上安装和配置TimescaleDB

本节向您展示如何在[支持的平台](#支持的平台)上使用Timescale提供的包安装最新版本的PostgreSQL和TimescaleDB。

<Tabs label="安装TimescaleDB">

<Tab title="Homebrew">

<SelfHostedHomebrew />

</Tab>

<Tab title="MacPorts">

<SelfHostedMacports />

</Tab>
</Tabs>

## 在数据库中添加TimescaleDB扩展

为了提高性能，您需要在自托管的PostgreSQL实例上的每个数据库上启用TimescaleDB。
本节向您展示如何使用命令行中的`psql`为PostgreSQL中的新数据库启用TimescaleDB。

<AddTimescaleDBToDB />

就是这样！您已经在自托管的PostgreSQL实例上的数据库上运行TimescaleDB。

## 接下来去哪里

<WhereTo />

## 支持的平台

为了获得最新功能，请安装macOS 14 Sanoma。支持的最旧版本是macOS 10.15 Catalina。

[install-postgresql]: https://www.postgresql.org/download/macosx/
[install-from-source]: https://www.postgresql.org/docs/current/source.html

[homebrew]: https://docs.brew.sh/Installation
[install-psql]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql/
[macports]: https://guide.macports.org/#installing.macports
[install-from-source]: /self-hosted/:currentVersion:/install/installation-source/
[install-postgresql]: https://www.postgresql.org/download/macosx/
