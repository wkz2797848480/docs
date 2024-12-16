---
标题: 多节点管理
摘要: 管理你的多节点 TimescaleDB 集群
产品: [自托管]
关键词: [多节点，管理]
标签: [管理]
---

import MultiNodeDeprecation from "versionContent/_partials/_multi-node-deprecation.mdx";

<MultiNodeDeprecation />

# 多节点管理

多节点TimescaleDB允许您直接从访问节点管理您的集群。当您的环境设置完成后，您不需要直接登录到数据节点来管理数据库。

当您执行管理任务时，例如添加新列、更改权限或在分布式超表上添加索引，您可以从访问节点执行任务，它将应用于所有数据节点。如果命令在常规表上执行，那么该命令的效果仅在访问节点上本地应用。同样，如果命令直接在数据节点上执行，结果仅在该数据节点上可见。

创建或修改模式、角色、表空间和设置的命令在分布式数据库中也不会自动分布。这是因为这些对象和设置有时需要在访问节点与数据节点之间不同，甚至在数据节点之间也有所不同。例如，数据节点可能具有独特的CPU、内存和磁盘配置。节点差异使得无法假设单一配置适用于所有节点。此外，某些设置需要在公开访问的访问节点与数据节点之间不同，例如具有不同的连接限制。一个角色可能在访问节点上没有`LOGIN`权限，但它需要在数据节点上具有此权限，以便访问节点可以连接。

角色和表空间也在同一实例上的多个数据库之间共享。这些数据库中有些可能是分布式的，有些可能不是，或者配置了不同的数据节点集。因此，当这些命令可以从需要不必分布式的不同数据库内执行时，无法确定何时应将角色或表空间分布到数据节点。

要从访问节点管理多节点集群，您可以使用[`distributed_exec`][distributed_exec]函数。此函数允许您完全控制跨所有数据节点创建和配置数据库设置、模式、角色和表空间。

本节的其余部分更详细地描述了在多节点环境中如何处理特定管理任务。

## 分布式角色管理

在多节点环境中，您需要独立管理每个PostgreSQL实例上的角色，因为角色是实例级对象，它们在分布式和非分布式数据库之间共享，每个数据库都可以配置不同的数据节点集或根本没有。因此，访问节点不会自动将角色或角色管理命令分布到其数据节点。当数据节点添加到集群时，假定它已经具有与其余节点一致所需的适当角色。如果不是这样，您可能在尝试创建或更改依赖于缺失或设置错误的的角色的对象时遇到意外错误。

为了帮助从访问节点管理角色，您可以使用[`distributed_exec`][distributed_exec]函数。这在当前数据库中跨所有数据节点创建和配置角色非常有用。

### 创建分布式角色

当您创建分布式角色时，重要的是要考虑同一角色在访问节点上可能需要与数据节点上不同的配置。例如，用户可能需要密码才能连接到访问节点，而集群内节点之间使用证书认证。您可能还想为外部连接设置连接限制，但允许与数据节点的内部连接无限制。例如，以下用户可以使用密码对访问节点进行10次连接，但连接到数据节点没有限制：

```sql
CREATE ROLE alice WITH LOGIN PASSWORD 'mypassword' CONNECTION LIMIT 10;
CALL distributed_exec($$ CREATE ROLE alice WITH LOGIN CONNECTION LIMIT -1; $$);
```

有关设置认证的更多信息，请参见[多节点认证部分][multi-node-authentication]。

有些角色也可以在访问节点上不使用`LOGIN`属性进行配置。这允许您在本地切换到该角色，但不能从远程位置使用该用户连接。然而，为了能够以该用户身份从访问节点连接到数据节点，数据节点需要将角色配置为启用`LOGIN`属性。要为多节点设置创建非登录角色，请使用以下命令：

```sql
CREATE ROLE alice WITHOUT LOGIN;
CALL distributed_exec($$ CREATE ROLE alice WITH LOGIN; $$);
```

要允许新角色创建分布式超表，它还需要在数据节点上被授予使用权，例如：

```sql
GRANT USAGE ON FOREIGN SERVER dn1,dn2,dn3 TO alice;
```

通过在某些数据节点上授予使用权，而不是其他节点，您可以根据角色限制使用到数据节点的子集。

### 更改分布式角色

当您更改分布式角色时，使用与创建角色相同的过程。角色需要在访问节点和数据节点上分两步更改。例如，向角色添加`CREATEROLE`属性如下：

```sql
ALTER ROLE alice CREATEROLE;
CALL distributed_exec($$ ALTER ROLE alice CREATEROLE; $$);
```

## 管理分布式数据库

分布式数据库可以包含分布式和非分布式对象。通常，当发出更改分布式对象的命令时，它适用于拥有该对象（或其部分）的所有节点。

然而，在某些情况下，根据节点不同设置*应该*不同，因为节点可能以不同的方式配置（例如，具有不同的CPU、内存和磁盘能力），访问节点的角色与数据节点的角色不同。

本节描述了当从分布式数据库内执行时，如何在所有数据节点上应用分布式对象的命令。

### 更改分布式数据库

[`ALTER DATABASE`][alter-database]命令仅在访问节点上本地应用。这是因为数据库级配置通常需要在节点间不同。例如，这是一个可能因节点CPU能力不同而不同的设置：

```sql
ALTER DATABASE mydatabase SET max_parallel_workers TO 12;
```

数据库名称在节点间也可以不同，即使数据库是同一分布式数据库的一部分。当您重命名数据节点的数据库时，还要确保更新访问节点上的数据节点配置，以便它引用新的数据库名称。

### 删除分布式数据库

当您在访问节点上删除分布式数据库时，它不会自动删除数据节点上相应的数据库。在这种情况下，您需要直接连接到每个数据节点并本地删除数据库。

分布式数据库不会自动跨所有节点删除，因为关于数据节点的信息存储在访问节点上的分布式数据库中，但在执行删除命令时无法读取它，因为在连接到数据库时无法发出删除命令。

此外，如果数据节点永久失败，您需要能够即使在一个或多个数据节点不响应的情况下也能删除数据库。

如果可能，最好保留数据节点上的数据。例如，即使在访问节点上删除了数据库，您可能仍希望备份数据节点。

或者，在访问节点上删除数据库之前，您可以使用`drop_database`选项删除数据节点：

```sql
SELECT * FROM delete_data_node('dn1', drop_database => true);
```

## 创建、更改和删除模式

当您创建、更改或删除模式时，这些命令不会自动跨所有数据节点应用。然而，当创建分布式超表且数据节点上不存在它所属的模式时，会创建缺少的模式。

要手动在所有数据节点上创建模式，请使用此命令：

```sql
CREATE SCHEMA newschema;
CALL distributed_exec($$ CREATE SCHEMA newschema $$);
```

如果模式是以特定授权创建的，那么在发出命令之前，授权角色也必须在数据节点上存在。更改现有模式的所有者也是如此。

### 使用DROP OWNED准备角色删除

[`DROP OWNED`][drop-owned]命令用于删除角色所拥有的所有对象，并为角色的删除做准备。执行以下命令，为分布式数据库中所有数据节点上的角色准备删除：

```sql
DROP OWNED BY alice CASCADE;
CALL distributed_exec($$ DROP OWNED BY alice CASCADE $$);
```

然而，请注意，在执行这些命令后，角色可能仍然拥有其他数据库中的对象。

### 管理权限

使用[`GRANT`][grant]或[`REVOKE`][revoke]语句配置的权限在分布式超表上运行时会应用于所有数据节点。在其他对象上授予权限时，需要使用[`distributed_exec`][distributed_exec]手动分发命令。

#### 设置默认权限

如果默认权限需要跨所有数据节点修改，需要使用[`distributed_exec`][distributed_exec]手动修改。默认权限引用的角色和模式需要在执行命令之前在数据节点上存在。

假定新数据节点已经具有任何更改的默认权限。默认权限不会自动追溯性地应用于新数据节点。

## 管理表空间

节点可能配置有不同的磁盘，因此需要在每个节点上手动配置表空间。特别是，访问节点可能与数据节点的存储配置不同，因为访问节点通常不存储大量数据。因此，不能假设多节点集群中的所有节点都存在相同的表空间配置。

[alter-database]: https://www.postgresql.org/docs/current/sql-alterdatabase.html 
[distributed_exec]: /api/:currentVersion:/distributed-hypertables/distributed_exec
[drop-owned]: https://www.postgresql.org/docs/current/sql-drop-owned.html 
[grant]: https://www.postgresql.org/docs/current/sql-grant.html 
[multi-node-authentication]: /self-hosted/:currentVersion:/multinode-timescaledb/multinode-auth/
[revoke]: https://www.postgresql.org/docs/current/sql-revoke.html
