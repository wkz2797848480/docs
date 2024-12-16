---
标题: 在 Linux 系统上安装 TimescaleDB
摘要: 在 Linux 系统上安装自托管的 TimescaleDB
产品: [自托管]
关键词: [安装，自托管，Debian（德比安），Ubuntu（乌班图），RHEL（红帽企业版 Linux），Fedora（费多拉）]
---

import WhereTo from "versionContent/_partials/_where-to-next.mdx";
import Skip from "versionContent/_partials/_selfhosted_cta.mdx";
import SelfHostedDebianBased from "versionContent/_partials/_install-self-hosted-debian-based.mdx";
import SelfHostedRedhatBased from "versionContent/_partials/_install-self-hosted-redhat-based.mdx";
import SelfHostedArchLinuxBased from "versionContent/_partials/_install-self-hosted-archlinux-based.mdx";
import AddTimescaleDBToDB from "versionContent/_partials/_add-timescaledb-to-a-database.mdx";


# 在 Linux 上安装 TimescaleDB

TimescaleDB 是一个 [PostgreSQL 扩展](https://www.postgresql.org/docs/current/external-extensions.html)，用于时间序列和高吞吐量的数据查询和摄入。

<跳过 />

本节展示了如何：

* [在 PostgreSQL 上安装和配置 TimescaleDB](#在-postgresql 上安装和配置-timescaledb) - 设置自托管的 PostgreSQL 实例以高效运行 TimescaleDB。
* [在数据库中添加 TimescaleDB 扩展](#在数据库中添加-timescaledb-扩展) - 在数据库上启用 TimescaleDB 特性和性能改进。

<Highlight type="warning">

如果您之前在没有包管理器的情况下安装了 PostgreSQL，您可能会在遵循这些安装说明时遇到错误。最佳实践是在开始之前完全移除任何现有的 PostgreSQL 安装。

要保留您当前的 PostgreSQL 安装，请[从源代码安装][install-from-source]。
</Highlight>

## 在 PostgreSQL 上安装和配置 TimescaleDB

本节展示了如何使用 TimeScale 提供的包在[支持的平台](#支持的平台)上安装最新版本的 PostgreSQL 和 TimescaleDB。

<Tabs label="安装 TimescaleDB">

<Tab title="Debian, Ubuntu">

<SelfHostedDebianBased />

</Tab>

<Tab title="Red Hat, Fedora">

<SelfHostedRedhatBased />

</Tab>

<Tab title="ArchLinux">

<SelfHostedArchLinuxBased />

</Tab>

</Tabs>

工作完成，您已经安装了 PostgreSQL 和 TimescaleDB。

## 在数据库中添加 TimescaleDB 扩展

为了提高性能，您需要在自托管的 PostgreSQL 实例上的每个数据库上启用 TimescaleDB。本节展示了如何使用命令行中的 `psql` 为 PostgreSQL 中的新数据库启用 TimescaleDB。

<添加 TimescaleDB 到数据库 />

就是这样！您已经在 PostgreSQL 的自托管实例上的数据库上运行了 TimescaleDB。

## 接下来做什么

<WhereTo />

## 支持的平台

TimescaleDB 支持以下平台：

| Debian | Ubuntu | Red Hat Enterprise | Fedora | Rocky Linux |
| - | - | - | - | - |
| Debian 10 Buster | Ubuntu 20.04 LTS Focal Fossa | Red Hat Enterprise Linux 7 | Fedora 33 | Rocky Linux 8 |
| Debian 11 Bullseye | Ubuntu 22.04 LTS Jammy Jellyfish | Red Hat Enterprise Linux 8 | Fedora 34 | Rocky Linux 9 |
| Debian 12 Bookworm | Ubuntu 23.04 Lunar Lobster | Red Hat Enterprise Linux 9 | Fedora 35 |  |


