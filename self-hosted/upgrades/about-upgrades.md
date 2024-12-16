---
标题: 关于升级
摘要: 关于主版本和次版本升级，以及升级的最佳实践。
产品: [自托管]
关键词: [升级]
---

import PlanUpgrade from "versionContent/_partials/_plan_upgrade.mdx";
import ExperimentalUpgrade from "versionContent/_partials/_experimental-schema-upgrade.mdx";
import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";

# 关于升级

**主要版本升级**是指从一个主要版本的TimescaleDB升级到下一个主要版本。例如，当您从TimescaleDB 1升级到TimescaleDB 2时。

**次要版本升级**是指在您当前的主要版本TimescaleDB内进行升级。例如，当您从TimescaleDB 2.5升级到TimescaleDB 2.6时。

如果您最初是使用Docker安装的TimescaleDB，您可以在Docker容器内进行升级。更多信息和指导，请参见[使用Docker升级部分][upgrade-docker]。

<ExperimentalUpgrade />

<ConsiderCloud />

## 规划您的升级

<PlanUpgrade />

<Highlight type="note">
如果您使用了Timescale Toolkit，请确保`timescaledb_toolkit`扩展版本为1.6.0，然后升级`timescaledb`扩展。如果需要，您可以随后将`timescaledb_toolkit`扩展升级到最新版本。
</Highlight>

## 检查您的版本

您可以在psql命令提示符下检查您正在运行的TimescaleDB版本。使用此命令在开始升级前检查您当前运行的版本，并在升级完成后再次检查：

```sql
\dx timescaledb

    Name     | Version |   Schema   |                             Description
-------------+---------+------------+---------------------------------------------------------------------
 timescaledb | x.y.z   | public     | Enables scalable inserts and complex queries for time-series data
(1 row)
```

[upgrade-docker]: /self-hosted/:currentVersion:/upgrades/upgrade-docker/
