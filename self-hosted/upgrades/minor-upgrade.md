---
标题: TimescaleDB 次版本升级
摘要: 将自托管的 TimescaleDB 升级到一个新的次版本。
产品: [自托管]
关键词: [升级]
---

import PlanUpgrade from "versionContent/_partials/_plan_upgrade.mdx";
import ConsiderCloud from "versionContent/_partials/_consider-cloud.mdx";
import CheckVersions from "versionContent/_partials/_migrate_self_postgres_check_versions.mdx";
import PlanMigrationPath from "versionContent/_partials/_migrate_self_postgres_plan_migration_path.mdx";
import ImplementMigrationPath from "versionContent/_partials/_migrate_self_postgres_implement_migration_path.mdx";

# 将 TimescaleDB 升级到新的次要版本

**次要版本升级**是指您从TimescaleDB `<主版本>.x`更新到TimescaleDB `<主版本>.y`。
**主要版本升级**是指您从TimescaleDB `X.<次要版本>`更新到`Y.<次要版本>`。
您可以在同一PostgreSQL实例的不同数据库上运行TimescaleDB的不同版本。
此过程使用PostgreSQL `ALTER EXTENSION`函数独立升级不同数据库上的TimescaleDB。

<ConsiderCloud />

本页向您展示如何执行次要版本升级，对于主要版本升级，请参阅[将TimescaleDB升级到主要版本][upgrade-major]。

## 先决条件

<PlanUpgrade />

## 检查TimescaleDB和PostgreSQL版本

<CheckVersions />

## 规划您的升级路径

<PlanMigrationPath />

## 实施您的升级路径

<ImplementMigrationPath />

您现在运行的是TimescaleDB的最新版本。

[relnotes]: https://github.com/timescale/timescaledb/releases 
[upgrade-pg]: /self-hosted/:currentVersion:/upgrades/upgrade-pg/#upgrade-postgresql
[upgrade-major]: /self-hosted/:currentVersion:/upgrades/major-upgrade/
[backup]: /self-hosted/:currentVersion:/backup-and-restore/
