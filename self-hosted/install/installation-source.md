---
标题: 从源代码安装 TimescaleDB
摘要: 从源代码安装自托管的 TimescaleDB
产品: [自托管]
关键词: [安装，自托管]
---

import WhereTo from "versionContent/_partials/_where-to-next.mdx";
import Skip from "versionContent/_partials/_selfhosted_cta.mdx";
import SelfHostedSource from "versionContent/_partials/_install-self-hosted-source-based.mdx";
import AddTimescaleDBToDB from "versionContent/_partials/_add-timescaledb-to-a-database.mdx";

# 从源代码自托管安装 TimescaleDB

TimescaleDB 是一个 [PostgreSQL 扩展](https://www.postgresql.org/docs/current/external-extensions.html)，用于时间序列和高吞吐量的数据查询和摄入。你可以从源代码在任何本地系统上安装 TimescaleDB 实例。

<跳过 />

本节展示了如何：

* [在 PostgreSQL 上安装和配置 TimescaleDB](#在-postgresql 上安装和配置-timescaledb) - 设置自托管的 PostgreSQL 实例以高效运行 TimescaleDB。
* [在数据库中添加 TimescaleDB 扩展](#在数据库中添加-timescaledb-扩展) - 在数据库上启用 TimescaleDB 特性和性能改进。

### 前提条件

要从源代码安装 TimescaleDB，你的开发环境中需要以下工具：

* **PostgreSQL**：

  根据 [PostgreSQL 安装说明][postgres-download] 安装一个[支持的 PostgreSQL 版本][compatibility-matrix]。

   我们建议不要使用 PostgreSQL 17.1、16.5、15.9、14.14、13.17、12.21 与 TimescaleDB 一起使用。
   这些次要版本[引入了一个破坏性的二进制接口变更][postgres-breaking-change]，这个变更在随后的次要版本 PostgreSQL 17.2、16.6、15.10、14.15、13.18 和 12.22 中被撤销。
   当你从源代码构建时，最佳实践是与 PostgreSQL 17.2、16.6 等更高版本一起构建。
   使用 [Timescale Cloud](https://console.cloud.timescale.com/) 和 Timescale 构建和分发的平台包的用户不受影响。

* **构建工具**：

  *   [CMake 版本 3.11 或更高][cmake-download]
  *   适用于您操作系统的 C 语言编译器，例如 `gcc` 或 `clang`。

      如果你使用的是 Microsoft Windows 系统，你可以安装 Visual Studio 2015 或更高版本代替 CMake 和 C 语言编译器。确保在运行安装程序时安装了 Visual Studio 的 CMake 和 Git 组件。

## 在 PostgreSQL 上安装和配置 TimescaleDB

本节展示了如何使用 Timescale 提供的源代码在支持的平台上安装最新版本的 PostgreSQL 和 TimescaleDB。

<SelfHostedSource />

## 在数据库中添加 TimescaleDB 扩展

为了提高性能，你需要在自托管的 PostgreSQL 实例上的每个数据库上启用 TimescaleDB。
本节展示了如何使用命令行中的 `psql` 为 PostgreSQL 中的新数据库启用 TimescaleDB。

<AddTimescaleDBToDB />

就是这样！你已经在 PostgreSQL 的自托管实例上的数据库上运行了 TimescaleDB。

## 接下来做什么

<WhereTo />

[install-psql]: /use-timescale/:currentVersion:/integrations/query-admin/about-psql/
[config]: /self-hosted/:currentVersion:/configuration/
[postgres-download]: https://www.postgresql.org/download/ 
[cmake-download]: https://cmake.org/download/ 
[compatibility-matrix]: /self-hosted/:currentVersion:/upgrades/upgrade-pg/#plan-your-upgrade-path
[postgres-breaking-change]: https://www.postgresql.org/about/news/postgresql-172-166-1510-1415-1318-and-1222-released-2965/
